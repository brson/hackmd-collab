# The Rust Compilation Model Calamity

## Or, Rust Compile-Time Adventures in TiKV: Part 1

---

_The Rust programming language was designed for slow compilation times._

This whole Rust thing is all a con the language designers played on you, the Rust user &mdash; adopt this crazy-fast, super-reliable language for your mission-critical products, and we'll slow your developers' productivity to a crawl.

Am I joking? I don't know. What even is anything anymore, anyway?

I do know one thing though…


## Rust compile times suck

If you are reading this you probably know it too. We all know it. We mostly hate it.

I know it because I encounter it every day. I know it because my coworkers complain about it. I know it because the Rust production users I talk to consistently mention it. I know it because so many of us indicated so [in the last Rust survey][sur].

[sur]: https://blog.rust-lang.org/2018/11/27/Rust-survey-2018.html#challenges

This is kinda infuriating, as almost everything that matters about Rust is pretty damn good. But Rust compile times are so, so bad.

Rust compile times are perhaps its biggest weakness.

At [PingCAP], we develop our distributed storage system, [TiKV], in Rust, and it compiles slow enough to discourage many in the company from using Rust. I recently spent some time ("some time" being months and months upon increasing months), along with several others on the TiKV team and its wider community, investigating TiKV's compile times.

[PingCAP]: https://pingcap.com/en/
[TiKV]: github.com/tikv/tikv/

Over a series of posts I'll discuss what we have learned (maybe &mdash; maybe I'll just leave you wondering forever): why compiling Rust is slow, and/or feels slow; compile-time use cases; things we measured; things we want to measure but haven't or don't know how; ideas that improved compile times; ideas that did not improve compile times; how TiKV compile times have changed over time; suggestions for how to organize Rust projects that compile fast; and recent and future upstream improvements to compile times.

In this episode:

- [The Spectre of Poor Rust Compile Times at PingCAP](#user-content-the-spectre-of-poor-rust-compile-times-at-pingcap)
- [Preview: The TiKV Compile-time Saga so far](#user-content-preview-the-tikv-compile-time-saga-so-far)
- [The Rust Compilation Model Calamity](#the-rust-compilation-model-calamity)
- [Bootstrapping Rust](#user-content-bootstrapping-rust)
- [(Un)virtuous cycles](#user-content-unvirtuous-cycles)
- [Recent work on Rust compile times](#user-content-recent-work-on-rust-compile-times)
- [In the next episode of The TiKV Compile-time Saga](#user-content-in-the-next-episode-of-the-tikv-compile-time-saga)
- [Thanks](#user-content-thanks)


## The Spectre of Poor Rust Compile Times at PingCAP

At [PingCAP], my colleagues write [TiKV], the storage node of our distributed database, in Rust. They do this because they want this most important node in the system to be fast and reliable by construction, at least to the greatest extent reasonable.

It was mostly a great decision, and most people internally are mostly happy about it.

But many complain about how long it takes to build. For some a full rebuild might take 15 minutes in development mode, and 30 minutes in release mode. To developers of large systems projects this might not sound horrible, but it's much slower than what many developers expect out of modern programming languages. TiKV is not even a particularly large system, with TODO total lines of Rust code. Building [Servo] or [Rust itself][r] is much, much more unpleasant.

[Servo]: https://github.com/servo/servo
[r]: github.com/rust-lang/rust/

Other nodes in the system are written in Go, which of course comes with a different set of advantages and disadvantages from Rust. Some of the Go developers at PingCAP kinda hate and resent having to wait for the Rust components to build. They are used to a rapid build-test cycle.

Rust developers on the other hand are used to taking a lot of coffee breaks. (Or tea, or cigarettes, or sobbing, or whatever, as the case may be &mdash; Rust developers have plenty of spare time to nurse their personal demons.)

_Internally at PingCAP, new code that would be appropriate to write in Rust is sometimes written in Go, only because of the spectre of terrible compile times_.

<!-- TODO: can the above be backed up with an example if asked? -->

This is a company that is one of Rust's greatest advocates in Asia.


## Preview: The TiKV Compile-time Saga so far

The first entry in this series is just a story about why Rust compile times suck. Since it might take another entry or two to dive into concrete technical details of what we've done with TiKV's compile times, here's a pretty graph to capture your imagination, without comment.

![tikv-compile-timings]

[tikv-compile-timings]: https://brson.github.io/tmp/tikv-timings.svg


## The Rust Compilation Model Calamity

Rust was designed for slow compilation times.

I mean, that wasn't _the goal_. As is often cautioned in debates among their designers, programming language design is full of tradeoffs. One of those fundamental tradeoffs is **run-time performance** vs. **compile-time performance**, and the Rust team nearly always (if not always) chose run-time over compile-time.

The intentional run-time / compile-time tradeoff isn't the only reason Rust compile times are horrific, but it's a big one. There are also language designs that are not crucial for run-time performance, but accidentally bad for compile time performance. The Rust compiler was also not designed to support fast compilation times.

So there are intrinsic language-design reasons, and accidental language-design reasons for Rust's bad compile times. Those mostly can't be fixed ever (though they may be mitigated by compiler improvements, design patterns, and language evolution). There are also accidental compiler-architecture reasons for Rust's bad compile times, which can generally be fixed through enormous engineering effort and time.

If compilation time was not a core Rust design principle, what were Rust's core design principles? Here are a few:

- Practicality &mdash; it should be a language that can be and is used in the real world.
- Pragmatism &mdash; it should admit concessions to human usability and integration into systems as they exist, instead of attempting to maintain any kind of theoretical purity.
- Memory-safety &mdash; it must enforce memory safety, and not admit segmentation faults and other such memory-access violations.
- Performance &mdash; it must be in the same performance class as C and C++.
- Concurrency &mdash; it must provide modern solutions to writing concurrent code.

But it's not like they didn't put _any_ consideration into fast compile times. For example, for any analysis Rust needs to do, the team tried to ensure it would not suffer from exponential complexity. Rust's design history though one of increasingly being sucked into the swamp of poor compile-time performance.

Story time.


## Bootstrapping Rust

I don't remember when I realized that Rust's bad compile times were a strategic problem for the language, potentially a fatal mistake in the face of future Rust-alike competitors. For the first few years I wasn't too concerned, and I don't think most of my peers were either. I mostly remember that Rust compile time was always* bad, and like, whatever, that's just reality.

<!-- TODO dcalvin doesn't like "and like, whatever" -->

When I worked on Rust heavily it was common for me to have at least three copies of the repository on the computer, hacking on one while all the others were building and testing. I would start building workspace 1, switch terminals, remember what's going on over here in workspace 2, hack on that for a while, start building in workspace 2, switch terminals, etc. Little flow, constant context switching.

This was (and probably is) typical of other Rust developers too.

(*) OK, Rust's compile time actually wasn't _always_ terrible. The [first Rust compiler][frc], called `rustboot`, written in OCaml, had extremely simple static analysis, and extremely naive code generation. Here's how long it takes to build:

```
TODO
```

That's only TODO minutes. For comparison here's a build of [today's compiler][tc] (as of December 2019):

```
TODO
```

Lol. Just lol. The comparison is completely unfair, for reasons. But lol, right? If you are curious, the full build logs for both runs are [yonder].

<!-- TODO: maybe a small gif here? -->

[frc]: https://github.com/rust-lang/rust/commit/d6b7c96c3eb29b9244ece0c046d3f372ff432d04
[tc]: https://github.com/rust-lang/rust/commit/3ed3b8bb7b100afecf7d5f52eafbb70fec27f537
[yonder]: todo

One of the immediate tasks upon Rust's initial open-sourcing in 2010 was to make it _self-hosting_ &mdash; that is, to write a Rust compiler in Rust. `rustc`, in addition to being written in Rust, would also use [LLVM] as its backend for generating machine code, instead of `rustboot`s hand-written x86 code-generator.

Rust needed to become self-hosting as a means of "dog-fooding" the language &mdash; writing the Rust compiler in Rust meant that the Rust authors needed to use their own language to write practical software, early in the language design process.

[LLVM]: https://llvm.org/

The first time Rust built itself was on April 20, 2011. [It took one hour][self-host], which was a laughably long time. At least it was back then.

[self-host]: https://mail.mozilla.org/pipermail/rust-dev/2011-April/000330.html

That first super-slow bootstrap was an anomaly of bad code-generation and other easily fixable early bugs (probably, I don't exactly recall). `rustc`'s performance quickly improved, and Graydon quickly [threw away the OCaml-`rustc`][nocaml] since there was nowhere near enough manpower and motivation to maintain parallel implementations. To compare with the previously-presented build times, that commit that drops OCaml bootstraps in TODO minutes under the same environment ([logs]).

[nocaml]: https://github.com/rust-lang/rust/commit/6997adf76342b7a6fe03c4bc370ce5fc5082a869
[logs]: todo

This is where the long, gruelling history of Rust's tragic compile times began, 11 months after it was initially released in June 2010.

---

Note that it is not actually fair to compare Rust's own compile time to the compile time of Rust programs generally, but it is fun because the Rust build takes so hilariously long.

Part of the reason for this is that Rust has to build itself 3 times to prove that it is self hosting. It does this in 3 "stages":

- _stage0_ - build from the previous release's binary. This proves that previous-Rust can build current-Rust.

- _stage1_ - build from current-Rust as built by previous-Rust. This proves that current-Rust can build current-Rust, but not that current-Rust-built-by-current-Rust works can build current-Rust (or next-Rust).

- _stage2_ - build from current-Rust as built by current-Rust. This proves that current-Rust can build a working current-Rust (or next-Rust), and that the self-hosting cycle will continue.

<!-- todo verify stages -->

The details of how Rust's stages interact is fairly complex, but this explains the basic reason Rust has to be compiled three times. These three stages are necessarily serialized, which does nothing to help its own compile time.


## (Un)virtuous cycles

In the Rust project we like processes that reinforce and build upon themselves. This is one of the keys to Rust's success, both as a language and community.

As an obvious, hugely-successful example, consider [Servo]. Servo is a web browser built in Rust (officially it's just a browser "engine"), and Rust was created with the explicit purpose of building Servo.

Rust and Servo are sister-projects. They were created by the same team (initially), at roughly the same time, and they evolved together. Not only was Rust built to create Servo, but Servo was built to inform the design of Rust.

The initial few years of both projects were extremely difficult, with both projects evolving in parallel. The often-used metaphor of the [Ship of Theseus][st] is apt: we were constantly rebuilding the ship we were sailing, constantly rebuilding Rust in order to sail the seas of Servo. There is no doubt that the experience of building Servo with Rust while simultaneously building Rust itself led directly to many of the good decisions that make Rust the practical language it is.

[st]: https://en.wikipedia.org/wiki/Ship_of_Theseus

Some examples of the Servo-Rust feedback loop,

- Labeled break and continue [was implemented in order to auto-generate an HTML parser][lbc].
- Owned closures [were implemented after analyzing closure usage in Servo][oc].
- Extern function calls used to be considered safe. [This changed in part due to experience in Servo][nu].
- The migration from green-threading to native threading was heavily informed by the experience of building Servo, observing the FFI overhead of Servo's SpiderMonkey integration, and profiling "hot splits", where the green thread stacks needed to be expanded and contracted.

[lbc]: https://github.com/rust-lang/rust/issues/2216
[oc]: https://github.com/rust-lang/rust/issues/2549#issuecomment-19588158
[nu]: https://github.com/rust-lang/rust/issues/2628#issuecomment-9384243

The co-development of Rust and Servo created a [virtuous cycle] that allowed both projects to thrive. Today, Servo components are deeply integrated into Firefox, ensuring that Rust cannot die while Firefox lives.

Mission accomplished.

[virtuous cycle]: https://en.wikipedia.org/wiki/Virtuous_circle_and_vicious_circle

The previously-mentioned early self-hosting was similarly crucial to Rust's design, making Rust a superior language for building Rust compilers. Likewise, Rust and [WebAssembly] were developed in close collaboration, making WASM an excellent platform for running Rust, and Rust the first language beyond C and C++ with decent WASM support.

[WebAssembly]: https://webassembly.org/

Sadly there was no such reinforcement to drive down compile times. The opposite is probably true: the more Rust became known as a "fast" language the more important it was to be "the fastest" language; and the more Rust's developers got used to developing their Rust projects across multiple branches, context switching between tasks as I described earlier, the less pressure was felt to address compile times.

That is, until Rust was actually released to production and met by a wide audince that was not used to such abhorrent compile times.

For years Rust [slowly boiled][boil] in its own poor compile times, not realizing how bad it had gotten until it was too late. It was 1.0. Those decisions were locked in. Rust was boiled.

[boil]: https://en.wikipedia.org/wiki/Boiling_frog

Too many metaphores in this section. Sorry.


## Recent work on Rust compile times

There is always work going on to improve Rust compile times. Here is a selection of the activity I'm aware of from the last year or two. Thanks to everybody who helps with this problem.

- The Rust issue tracker has a [master issue] for tracking compile-time-related work.
- Pipelined compilation ([1][pipe1], [2][pipe2], [3][pipe3])
  - Typechecks downstream crates in parallel with upstream codegen. Now on by default on the stable channel.
  - Developed by [@alexcrichton] and [@nikomatsakis].
- Parallel rustc ([1][parc1], [2][parc2], [3][parc3])
  - Runs analysis phases of the compiler in parallel. Not yet availble on the stable channel.
  - Developed by [@Zoxc], [@michaelwoerister], [@oli-obk], and others.
- `cargo build -Ztimings` ([1][cbt1], [2][cbt2])
  - Collects and graphs information about cargo's parallel build timings.
  - Developed by [@ehuss] and [@luser].
- `rustc -Zself-profile` ([1][sp1], [2][sp2], [3][sp3])
  - Generates detailed information about `rustc`'s internal performance.
  - Developed by [@wesleywiser] and [@michaelwoerister].
- [cargo-feature-analyst]
  - Finds unused features.
  - Developed by [@psinghal20].
- [cargo-udeps]
  - Finds unused crates.
  - Developed by [@est31].
- [twiggy]
  - Profiles code size, which is corellated with compile time.
  - Developed by [@fitzgen], [@data-pup], and others.

[master issue]: https://github.com/rust-lang/rust/issues/48547
[twiggy]: https://github.com/rustwasm/twiggy
[@fitzgen]: https://github.com/fitzgen
[@data-pup]: https://github.com/data-pup
[cbt1]: https://internals.rust-lang.org/t/exploring-crate-graph-build-times-with-cargo-build-ztimings/10975
[cbt2]: https://github.com/rust-lang/cargo/issues/7405
[@ehuss]: https://github.com/ehuss
[@luser]: https://github.com/luser
[pipe1]: https://github.com/rust-lang/rust/issues/60988
[pipe2]: https://github.com/rust-lang/cargo/issues/6660
[pipe3]: https://internals.rust-lang.org/t/evaluating-pipelined-rustc-compilation/10199
[@alexcrichton]: https://github.com/alexcrichton
[@nikomatsakis]: https://github.com/nikomatsakis
[parc1]: https://internals.rust-lang.org/t/parallelizing-rustc-using-rayon/6606
[parc2]: https://github.com/rust-lang/rust/issues/48685
[parc3]: https://internals.rust-lang.org/t/help-test-parallel-rustc/11503/14
[@Zoxc]: https://github.com/Zoxc
[@michaelwoerister]: https://github.com/michaelwoerister
[@oli-obk]: http://github.com/oli-obk
[sp1]: https://rust-lang.github.io/rustc-guide/profiling.html
[sp2]: https://github.com/rust-lang/rust/issues/58967
[sp3]: https://github.com/rust-lang/rust/pull/51657
[@wesleywiser]: https://github.com/wesleywiser
[cargo-feature-analyst]: https://github.com/psinghal20/cargo-feature-analyst
[@psinghal20]: https://github.com/psinghal20
[cargo-udeps]: https://github.com/est31/cargo-udeps
[@est31]: https://github.com/est31

- https://github.com/rust-analyzer
- https://github.com/RazrFalcon/cargo-bloat
- https://github.com/rust-lang/rust/issues/58967
- https://vfoley.xyz/rust-compile-speed-tips/
- https://doc.rust-lang.org/nightly/cargo/reference/unstable.html#profile-overrides
- https://raphlinus.github.io/rust/2019/08/21/rust-bloat.html
- https://blog.mozilla.org/nnethercote/2019/10/11/how-to-speed-up-the-rust-compiler-some-more-in-2019/

I apologize to person or project I didn't credit.

Finally, here are some tools and scripts I wrote in the course of my TiKV compile-time work. These
are not polished, but may prove useful to others, at least conceptually. Pull requests welcome.

  - https://github.com/brson/bench-cargo-profiles
  - https://github.com/brson/cargo-bloat/tree/generics-collapse
  - https://github.com/brson/tikv-bench-scripts
  - https://github.com/brson/measure-rustc-rss
  - maptime
  - time-all-profiles.sh
    - https://gist.github.com/brson/1547c3315739440c0c3aef1dc44e0ee4
  - sum-time-passes.py
    - https://gist.github.com/brson/819c52c6e9f09f5eaa450e623c686e4e


## In the next episode of The TiKV Compile-time Saga

Things are looking dire for Rust developers' productivity, and TiKV hackers are grumbling during their frequent coffee breaks! Can Rust succeed? Can Rust compile TiKV fast enough to prevent PingCAP's product managers from &nbsp; (╯°□°）╯︵&nbsp;┻━┻ &nbsp; and rewriting the entire thing in C++ or Go or Pony?

In the next entry of this series we'll talk about the nuances of the many Rust compile time use cases, which aspects of Rust affect which use cases, how users percieve and are affected by Rust compile time, and all the ideas the TiKV team came up with to free ourselves from this Rusting, suffering life.

Stay Rusty, friends.


## Thanks

This post has been in draft for a long time, and has benefited from the input and review of several people. Thanks especially to Calvin Weng for the reviews. Thanks to Niko Matsakis for advice about Rust's compile time behavior. Thanks to Graydon Hoare for recollections about Rust's design. Thanks to others for their patience about my extreme procrastination.

