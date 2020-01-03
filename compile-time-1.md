# The Rust Compilation Model Calamity

## Or, Rust Compile-time Adventures in TiKV: Part 1

---

The Rust programming language was designed for slow compilation times.

I was there. I witnessed it for myself, and I am finally ready to break the silence: Rust is a con. It's a prank the language designers played on you, the Rust user &mdash; adopt this crazy-fast, super-reliable language for your products, and we'll reduce your developers' productivity to a crawl.

**WE GOT YOU GOOD!**

![taunting crab](https://brson.github.io/tmp/crab1.jpg)
<!--![taunting crab](https://brson.github.io/tmp/crab2.jpg)-->
<!--![taunting crab](https://brson.github.io/tmp/crab3.jpg)-->
<!--![taunting crab](https://brson.github.io/tmp/crab4.jpg)-->
<!--![taunting crab](https://brson.github.io/tmp/crab5.jpg)-->

Am I joking? I don't know. What is reality anyway?

I do know one thing though…


## Rust compile times suck

If you are reading this you probably know it too.

I know it because I encounter it every day. I know it because my coworkers complain about it. I know it because the Rust production users I talk to consistently mention it. I know it because so many of us indicated so [in the last Rust survey][sur].

[sur]: https://blog.rust-lang.org/2018/11/27/Rust-survey-2018.html#challenges

This is kinda infuriating, as almost everything that matters about Rust is pretty damn good. But Rust compile times are so, so bad.

Rust compile times are perhaps its biggest weakness.

At [PingCAP], we develop our distributed storage system, [TiKV], in Rust, and it compiles slow enough to discourage many in the company from using Rust. I recently spent some time ("some time" being months and months upon increasing months), along with several others on the TiKV team and its wider community, investigating TiKV's compile times.

[PingCAP]: https://pingcap.com/en/
[TiKV]: github.com/tikv/tikv/

Over a series of posts I'll discuss what we have learned: why compiling Rust is slow, and/or feels slow; how Rust's development lead to slow compile-times; compile-time use cases; things we measured; things we want to measure but haven't or don't know how; ideas that improved compile times; ideas that did not improve compile times; how TiKV compile times have changed over time; suggestions for how to organize Rust projects that compile fast; and recent and future upstream improvements to compile times.

In this episode:

- [The spectre of poor Rust compile times at PingCAP](#user-content-the-spectre-of-poor-rust-compile-times-at-pingcap)
- [Preview: The TiKV compile-time adventure so far](#user-content-preview-the-tikv-compile-time-saga-so-far)
- [The Rust compilation model calamity](#the-rust-compilation-model-calamity)
- [Bootstrapping Rust](#user-content-bootstrapping-rust)
- [(Un)virtuous cycles](#user-content-unvirtuous-cycles)
- [Early design decisions that favored run-time over compile-time](#user-content-early-decisions-that-favored-run-time-over-compile-time)
- [Recent work on Rust compile times](#user-content-recent-work-on-rust-compile-times)
- [In the next episode](#user-content-in-the-next-episode-of-the-tikv-compile-time-saga)
- [Addendum: Thanks](#user-content-addendum-thanks)
- [Addendum: Bad metaphore body-count](#user-content-addendum-bad-metaphore-body-count)


## The spectre of poor Rust compile times at PingCAP

At [PingCAP], my colleagues write [TiKV], the storage node of our distributed database, in Rust. They do this because they want this most important node in the system to be fast and reliable by construction, at least to the greatest extent reasonable.

It was mostly a great decision, and most people internally are mostly happy about it.

But many complain about how long it takes to build. For some a full rebuild might take 15 minutes in development mode, and 30 minutes in release mode. To developers of large systems projects this might not sound horrible, but it's much slower than what many developers expect out of modern programming environments. TiKV is not even a particularly large system, with 2 million total lines of Rust code (TODO and other code? TODO not large?). Building [Servo] or [Rust itself][r] is much, much more unpleasant.

[Servo]: https://github.com/servo/servo
[r]: github.com/rust-lang/rust/

Other nodes in the system are written in Go, which of course comes with a different set of advantages and disadvantages from Rust. Some of the Go developers at PingCAP resent having to wait for the Rust components to build. They are used to a rapid build-test cycle.

Rust developers on the other hand are used to taking a lot of coffee breaks (or tea, or cigarettes, or sobbing, or whatever, as the case may be &mdash; Rust developers have the spare time to nurse their demons).

TODO hook


## Preview: The TiKV Compile-time adventure so far

The first entry in this series is just a story about the history of Rust with respect to compilation time. Since it might take several more entries before we dive into concrete technical details of what we've done with TiKV's compile times, here's a pretty graph to capture your imagination, without comment.

![tikv-compile-timings]

[tikv-compile-timings]: https://brson.github.io/tmp/tikv-timings.svg


## The Rust Compilation Model Calamity

Rust was designed for slow compilation times.

I mean, that wasn't _the goal_. As is often cautioned in debates among their designers, programming language design is full of tradeoffs. One of those fundamental tradeoffs is **run-time performance** vs. **compile-time performance**, and the Rust team nearly always (if not always) chose run-time over compile-time.

The intentional run-time / compile-time tradeoff isn't the only reason Rust compile times are horrific, but it's a big one. There are also language designs that are not crucial for run-time performance, but accidentally bad for compile-time performance. The Rust compiler was also implemented in ways that inhibit compile-time performance.

So there are intrinsic language-design reasons, and accidental language-design reasons for Rust's bad compile times. Those mostly can't be fixed ever (though they may be mitigated by compiler improvements, design patterns, and language evolution). There are also accidental compiler-architecture reasons for Rust's bad compile times, which can generally be fixed through enormous engineering effort and time.

If fast compilation time was not a core Rust design principle, what were Rust's core design principles? Here are a few:

- Practicality &mdash; it should be a language that can be and is used in the real world.
- Pragmatism &mdash; it should admit concessions to human usability and integration into systems as they exist today.
- Memory-safety &mdash; it must enforce memory safety, and not admit segmentation faults and other such memory-access violations.
- Performance &mdash; it must be in the same performance class as C++.
- Concurrency &mdash; it must provide modern solutions to writing concurrent code.

But it's not like the designers didn't put _any_ consideration into fast compile times. For example, for any analysis Rust needs to do, the team tried to ensure reasonable bounds on computational complexity. Rust's design history though one of increasingly being sucked into the swamp of poor compile-time performance.

Story time.


## Bootstrapping Rust

I don't remember when I realized that Rust's bad compile times were a strategic problem for the language, potentially a fatal mistake in the face of competition from future low-level programming languages. For the first few years, hacking almost entirely on the Rust compiler itself, I wasn't too concerned, and I don't think most of my peers were either. I mostly remember that Rust compile time was always* bad, and like, whatever, I can deal with that.

When I worked daily on the Rust compiler it was common for me to have at least three copies of the repository on the computer, hacking on one while all the others were building and testing. I would start building workspace 1, switch terminals, remember what's going on over here in workspace 2, hack on that for a while, start building in workspace 2, switch terminals, etc. Little flow, constant context switching.

This was (and probably is) typical of other Rust developers too. I still do the same thing hacking on TiKV today.

---

\* OK, Rust's (self-)compile time actually wasn't _always_ terrible. The [first Rust compiler][frc], called `rustboot`, written in OCaml, had extremely simple static analysis, and extremely naive code generation. Here's how long it takes to build:

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

---

One of the immediate tasks upon Rust's initial open-sourcing in 2010 was to make it _self-hosting_ &mdash; that is, to write a Rust compiler in Rust. `rustc`, in addition to being written in Rust, would also use [LLVM] as its backend for generating machine code, instead of `rustboot`s hand-written x86 code-generator.

Rust needed to become self-hosting as a means of "dog-fooding" the language &mdash; writing the Rust compiler in Rust meant that the Rust authors needed to use their own language to write practical software, early in the language design process, which it was hoped would lead to a useful and practical language.

[LLVM]: https://llvm.org/

The first time Rust built itself was on April 20, 2011. [It took one hour][self-host], which was a laughably long time. At least it was back then.

[self-host]: https://mail.mozilla.org/pipermail/rust-dev/2011-April/000330.html

That first super-slow bootstrap was an anomaly of bad code-generation and other easily fixable early bugs (probably, I don't exactly recall). `rustc`'s performance quickly improved, and Graydon quickly [threw away the old `rustboot` compiler][nocaml] since there was nowhere near enough manpower and motivation to maintain parallel implementations. To compare with the previously-presented build times, that commit that drops OCaml bootstraps in TODO minutes under the same environment ([logs]).

[nocaml]: https://github.com/rust-lang/rust/commit/6997adf76342b7a6fe03c4bc370ce5fc5082a869
[logs]: todo

This is where the long, gruelling history of Rust's tragic compile times began, 11 months after it was initially released in June 2010.

<!--

- init: https://gist.github.com/a6b599da4e1a7faa177c707baa095caf
- bootstrap:
- today:

-->

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

Too many tired metaphores in this section. Sorry, no time to edit them into something more creative. Deadlines expiring.


## Early design decisions that favored run-time over compile-time

What follows is a brief description of some of Rust's key early design decisions that contribute to slow compile times. I only describe them briefly here. The next episode in this series will go into further depth.

- Borrowing

- Monomorphization

- Stack unwinding

- Macros

- LLVM backend

- Relying too much on the LLVM optimizer

- Split compiler / package manager

- Per-compilation-unit code-generation

- Single-threaded compiler

- Traits and trait coherence

- Tests in same compilation unit as code


## Recent work on Rust compile times

There is always work going on to improve Rust compile times. Here is a selection of the activity I'm aware of from the last year or two. Thanks to everybody who helps with this problem.

- The Rust compile-time [master issue]
  - Tracks various work to improve compile times.
  - Contains a great overview of factors that affect Rust compilation performance and potential mitigation strategies.
- Pipelined compilation ([1][pipe1], [2][pipe2], [3][pipe3])
  - Typechecks downstream crates in parallel with upstream codegen. Now on by default on the stable channel.
  - Developed by [@alexcrichton] and [@nikomatsakis].
- Parallel rustc ([1][parc1], [2][parc2], [3][parc3])
  - Runs analysis phases of the compiler in parallel. Not yet availble on the stable channel.
  - Developed by [@Zoxc], [@michaelwoerister], [@oli-obk], and others.
- [MIR-level constant propagation][cprop]
  - Performs constant propagation on MIR, which reduces duplicated LLVM work on monomorphized functions.
  - Developed by [@wesleywiser].
- [MIR optimizations][mo1]
  - Optimizing MIR should be faster that optimizeng monomorphized LLVM IR.
  - Not in stable compilers yet.
  - Developed by [@wesleywiser] and others.
- `cargo build -Ztimings` ([1][cbt1], [2][cbt2])
  - Collects and graphs information about cargo's parallel build timings.
  - Developed by [@ehuss] and [@luser].
- `rustc -Zself-profile` ([1][sp1], [2][sp2], [3][sp3])
  - Generates detailed information about `rustc`'s internal performance.
  - Developed by [@wesleywiser] and [@michaelwoerister].
- [Shared monomorphizations][sm]
  - Reduces code bloat by deduplicating monomorphizations that occur in multiple crates.
  - Enabled by default if the optimization level is less than 3.
  - Developed by [@michaelwoerister].
- [perf.rust-lang.org]
  - Rust's compile-time performance is tracked in detail. Benchmarks continue to be added.
  - Developed by [@nrc], [@Mark-Simulacrum], [@nnethercote] and many more.
- [cargo-bloat]
  - Finds what occupies the most space in binaries. Bloat is correlated with compile time.
  - Developed by [@RazrFalcon] and others.
- [cargo-feature-analyst]
  - Finds unused features.
  - Developed by [@psinghal20].
- [cargo-udeps]
  - Finds unused crates.
  - Developed by [@est31].
- [twiggy]
  - Profiles code size, which is corellated with compile time.
  - Developed by [@fitzgen], [@data-pup], and others.
- [rust-analyzer]
  - A new language server for Rust with faster response time than the original [RLS].
  - Developed by [@matklad], [@flodiebold], [@kjeremy], and many others.
- ["How to alleviate the pain of Rust compile times"][ctpain]
  - Blog post by vfoley.
- ["Thoughts on Rust bloat"][trb]
  - Blog post by [@raphlinus]
- Nicholas Nethercote's work on `rustc` optimization
  - ["How to speed up the Rust compiler in 2019"][nn1]
  - ["The Rust compiler is still getting faster"][nn2]
  - ["Visualizing Rust compilation"][nn3]
  - ["How to speed up the Rust compiler some more in 2019"][nn4]
  - ["How to speed up the Rust compiler one last time in 2019"][nn5]

[mo1]: https://github.com/rust-lang/rust/pulls?q=mir-opt
[nn5]: https://blog.mozilla.org/nnethercote/2019/12/11/how-to-speed-up-the-rust-compiler-one-last-time-in-2019/
[nn4]: https://blog.mozilla.org/nnethercote/2019/10/11/how-to-speed-up-the-rust-compiler-some-more-in-2019/
[nn3]: https://blog.mozilla.org/nnethercote/2019/10/10/visualizing-rust-compilation/
[nn2]: https://blog.mozilla.org/nnethercote/2019/07/25/the-rust-compiler-is-still-getting-faster/
[nn1]: https://blog.mozilla.org/nnethercote/2019/07/17/how-to-speed-up-the-rust-compiler-in-2019/
[trb]: https://raphlinus.github.io/rust/2019/08/21/rust-bloat.html
[@raphlinus]: https://github.com/raphlinus
[ctpain]: https://vfoley.xyz/rust-compile-speed-tips/
[cargo-bloat]: https://github.com/RazrFalcon/cargo-bloat
[@RazrFalcon]: https://github.com/RazrFalcon
[perf.rust-lang.org]: https://perf.rust-lang.org/
[@Mark-Simulacrum]: https://github.com/Mark-Simulacrum
[@nrc]: https://github.com/nrc
[@nnethercote]: https://github.com/nnethercote
[cprop]: https://blog.rust-lang.org/inside-rust/2019/12/02/const-prop-on-by-default.html
[sm]: https://github.com/rust-lang/rust/issues/47317
[rust-analyzer]: https://github.com/rust-analyzer/rust-analyzer
[@matklad]: https://github.com/matklad
[@flodiebold]: https://github.com/flodiebold
[@kjeremy]: https://github.com/kjeremy
[RLS]: https://github.com/rust-lang/rls
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

I apologize to any person or project I didn't credit.


## In the next episode of Rust Compile-time Adventures in TiKV

So Rust dug itself deep into a corner over the years and will probably be digging itself back out until the end of time (or the end of Rust &mdash; same thing, really). Can Rust compile-time be saved from its own run-time success? Will TiKV ever build fast enough to satisfy my managers?

In the next episode, we'll deep-dive into the specifics of Rust's language design that cause it to compile so slow.

Stay Rusty, friends.


## Addendum: Thanks

This work has benefited from the input and review of several people. Thanks especially to Calvin Weng for the reviews. Thanks to Niko Matsakis for advice about Rust's compile time behavior. Thanks to Graydon Hoare for recollections about Rust's design. Thanks to others for their patience.


## Addendum: Bad metaphore body-count

Let's see how many cliché's I managed to bake into this one:

- clickbait title
- pretentious subtitle
- stupid "meme" imagery
- "sucked into the swamp of poor compile-time performance"
- "dog-fooding"
- "gruelling history of Rust's tragic compile times"
- The Ship of Theseus
- virtuous cycles
- "mission accomplished"
- "for years Rust slowly boiled"
- "dug deep into a corner" (two in one!)
- TV-style cliffhanger ending
- "bake into this one"

I'm sorry. I'm sorry. I don't care. Leave me alone.


<!--

images:

- https://www.pixel4k.com/crab-4k-69365.html
- http://yesofcorsa.com/blue-crab/
- https://commons.wikimedia.org/wiki/File:K%C4%B1z%C4%B1l%C4%B1rmak_near_the_crab.jpg
- https://cdn4.gamepur.com/images/god-of-war/Mr_Krabs_And_Money.jpg
- http://nick.mtvnimages.com/nick/video/images/spongebob-squarepants/spongebob-squarepants-mr-krabs-formula-clip-16x9.jpg?quality=0.75&maxdimension=600&height=225&width=400

-->