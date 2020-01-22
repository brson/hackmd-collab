# The Rust Compilation Model Calamity

## Or, Rust Compile-time Adventures in TiKV: Part 1

---

The Rust programming language was designed for slow compilation times.

I was there. I witnessed it for myself, and I am finally ready to break the silence: Rust is a hoax. It's a prank the language designers played on you, the Rust user &mdash; adopt this high-performance, high-reliability language for your products, and we'll reduce your productivity to a crawl.

It's brilliantly insidious.

&nbsp;

&#x1f942; **WE GOT YOU GOOD!** &#x1f942;

![taunting crab](https://brson.github.io/tmp/crab1.jpg)

&nbsp;

Think I'm joking? It doesn't matter. What is reality anyway?

One thing that does matter?

&nbsp;


## Rust compile times are bad

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


## The spectre of poor Rust compile times at PingCAP

At [PingCAP], my colleagues write [TiKV], the storage node of our distributed database, in Rust. They do this because they want this most important node in the system to be fast and reliable by construction, at least to the greatest extent reasonable.

It was mostly a great decision, and most people internally are mostly happy about it.

But many complain about how long it takes to build. For some a full rebuild might take 15 minutes in development mode, and 30 minutes in release mode. To developers of large systems projects this might not sound horrible, but it's much slower than what many developers expect out of modern programming environments. (I wanted to say here that TiKV is not even that big of a codebase but it does contain 2 million lines of Rust and 2 million lines of C/C++, which is sizable. In comparison, Rust itself contains over 3 million lines of Rust, and [Servo] contains 2.7 million. [Full line counts here]).

[Servo]: https://github.com/servo/servo
[Full line counts here]: https://gist.github.com/brson/31b6f8c5467b050779ce9aa05d41aa84/edit

Other nodes in the system are written in Go, which of course comes with a different set of advantages and disadvantages from Rust. Some of the Go developers at PingCAP resent having to wait for the Rust components to build. They are used to a rapid build-test cycle.

Rust developers on the other hand are used to taking a lot of coffee breaks (or tea, or cigarettes, or sobbing, or whatever, as the case may be &mdash; Rust developers have the spare time to nurse their demons).


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

- _Practicality_ &mdash; it should be a language that can be and is used in the real world.
- _Pragmatism_ &mdash; it should admit concessions to human usability and integration into systems as they exist today.
- _Memory-safety_ &mdash; it must enforce memory safety, and not admit segmentation faults and other such memory-access violations.
- _Performance_ &mdash; it must be in the same performance class as C++.
- _Concurrency_ &mdash; it must provide modern solutions to writing concurrent code.

But it's not like the designers didn't put _any_ consideration into fast compile times. For example, for any analysis Rust needs to do, the team tried to ensure reasonable bounds on computational complexity. Rust's design history though one of increasingly being sucked into the swamp of poor compile-time performance.

Story time.


## Bootstrapping Rust

I don't remember when I realized that Rust's bad compile times were a strategic problem for the language, potentially a fatal mistake in the face of competition from future low-level programming languages. For the first few years, hacking almost entirely on the Rust compiler itself, I wasn't too concerned, and I don't think most of my peers were either. I mostly remember that Rust compile time was always* bad, and like, whatever, I can deal with that.

When I worked daily on the Rust compiler it was common for me to have at least three copies of the repository on the computer, hacking on one while all the others were building and testing. I would start building workspace 1, switch terminals, remember what's going on over here in workspace 2, hack on that for a while, start building in workspace 2, switch terminals, etc. Little flow, constant context switching.

This was (and probably is) typical of other Rust developers too. I still do the same thing hacking on TiKV today.

---

\* So, historically, how bad have Rust compile times been? A simple barometer here is to see how Rust's self-hosting times have changed over the years, that is the time it takes Rust to build itself. Rust building itself is not directly comparable to Rust building other projects, for a variety of reasons, but I think it will be illustrative.

The [first Rust compiler][frc], from 2010, called `rustboot`, was written in `OCaml`, and it's ultimate purpose was to build a second compiler, `rustc`, written in Rust, and begin the self-hosting bootstrap cycle.

[frc]: https://gist.github.com/brson/31b6f8c5467b050779ce9aa05d41aa84/edit

`rustc`, in addition to being written in Rust, would also use [LLVM] as its backend for generating machine code, instead of `rustboot`s hand-written x86 code-generator.

Rust needed to become self-hosting as a means of "dog-fooding" the language &mdash; writing the Rust compiler in Rust meant that the Rust authors needed to use their own language to write practical software, early in the language design process, which it was hoped would lead to a useful and practical language.

[LLVM]: https://llvm.org/

The first time Rust built itself was on April 20, 2011. [It took one hour][self-host], which was a laughably long time. At least it was back then.

[self-host]: https://mail.mozilla.org/pipermail/rust-dev/2011-April/000330.html

That first super-slow bootstrap was an anomaly of bad code-generation and other easily fixable early bugs (probably, I don't exactly recall). `rustc`'s performance quickly improved, and Graydon quickly [threw away the old `rustboot` compiler][nocaml] since there was nowhere near enough manpower and motivation to maintain parallel implementations.

This is where the long, gruelling history of Rust's tragic compile times began, 11 months after it was initially released in June 2010.

Thesis: The Rust language developers became acclimated to Rust's poor self-hosting times and failed to recognize the severity of the problem of bad compile times during Rust's crucial early design phase.

_Note: I wanted to share historic self-hosting times here to support the above thesis, but after many hours and obstactles attempting to build Rust revisions from 2011, I finally gave up and decided I just had to publish this piece without. Instead here are some made up numbers:_

- _7 femto-bunnies_ - `rustboot` building Rust prior to being retired
- _49 kilo-hamsters_ - `rustc` building Rust immidately after `rustboot`s retirement
- _188 giga-sloths_ - `rustc` building Rust in 2020

Anyway, last time I bootstrapped Rust a few months ago it took over five hours.


## (Un)virtuous cycles

In the Rust project we like processes that reinforce and build upon themselves. This is one of the keys to Rust's success, both as a language and community.

As an obvious, hugely-successful example, consider [Servo]. Servo is a web browser built in Rust, and Rust was created with the explicit purpose of building Servo. Rust and Servo are sister-projects. They were created by the same team (initially), at roughly the same time, and they evolved together. Not only was Rust built to create Servo, but Servo was built to inform the design of Rust.

The initial few years of both projects were extremely difficult, with both projects evolving in parallel. The often-used metaphor of the [Ship of Theseus][st] is apt: we were constantly rebuilding the ship we were sailing, constantly rebuilding Rust in order to sail the seas of Servo. There is no doubt that the experience of building Servo with Rust while simultaneously building Rust itself led directly to many of the good decisions that make Rust the practical language it is.

[st]: https://en.wikipedia.org/wiki/Ship_of_Theseus

Some examples of the Servo-Rust feedback loop,

- Labeled break and continue [was implemented in order to auto-generate an HTML parser][lbc].
- Owned closures [were implemented after analyzing closure usage in Servo][oc].
- Extern function calls used to be considered safe. [This changed in part due to experience in Servo][nu].
- The migration from green-threading to native threading was informed by the experience of building Servo, observing the FFI overhead of Servo's SpiderMonkey integration, and profiling "hot splits", where the green thread stacks needed to be expanded and contracted.

[lbc]: https://github.com/rust-lang/rust/issues/2216
[oc]: https://github.com/rust-lang/rust/issues/2549#issuecomment-19588158
[nu]: https://github.com/rust-lang/rust/issues/2628#issuecomment-9384243

The co-development of Rust and Servo created a [virtuous cycle] that allowed both projects to thrive. Today, Servo components are deeply integrated into Firefox, ensuring that Rust cannot die while Firefox lives.

[virtuous cycle]: https://en.wikipedia.org/wiki/Virtuous_circle_and_vicious_circle

&nbsp;

Mission accomplished.

![astronaut planting rust flag on the moon](https://brson.github.io/tmp/moonflag2.jpg)

&nbsp;

The previously-mentioned early self-hosting was similarly crucial to Rust's design, making Rust a superior language for building Rust compilers. Likewise, Rust and [WebAssembly] were developed in close collaboration (the author of Emscripten and I had desks next to each other for years), making WASM an excellent platform for running Rust, and Rust well-suited to target WASM.

[WebAssembly]: https://webassembly.org/

Sadly there was no such reinforcement to drive down Rust compile times. The opposite is probably true: the more Rust became known as a _fast_ language the more important it was to be _the fastest_ language; and the more Rust's developers got used to developing their Rust projects across multiple branches, context switching between builds, the less pressure was felt to address compile times.

That is, until Rust was actually released to production and met by a wide audince that was not so tolerant of slow compile times.

For years Rust [slowly boiled][boil] in its own poor compile times, not realizing how bad it had gotten until it was too late. It was 1.0. Those decisions were locked in. Rust was boiled.

[boil]: https://en.wikipedia.org/wiki/Boiling_frog

Too many tired metaphores in this section. Sorry, no time to edit them into something more creative. Deadlines expiring.


## Early design decisions that favored run-time over compile-time

If Rust is designed for poor compile time, then what are those designs specifically?. I describe a few briefly here. The next episode in this series will go into further depth. Some have greater compile-time impact than others, but I assert that all of them cause more time to be spent in compilation than alternative designs.

Looking at some of these in retrospect it's tempting to think that "well, of course Rust _must_ have feature _foo_", and it's true that Rust would be a completely different language without many of these features; but language designs are tradeoffs and none of these were predestined to be part of Rust.

- _Borrowing_ &mdash; Rust's defining feature &mdash; its sophisticated pointer analysis &mdash; spends compile-time to make run-time safe.

- _Monomorphization_ &mdash; Rust translates each generic instantiation into its own machine code, creating code bloat and increasing compile time.

- _Stack unwinding_ &mdash; stack unwinding after unrecoverable exceptions traverses the callstack backwards and runs cleanup code. It requires lots of compile-time book-keeping and code generation.

- _Build scripts_ &mdash; build scripts allow arbitrary code to be run at compile-time, and pull in their own dependencies that need to be compiled. Their unknown side-effects and unknown inputs and outputs limit assumptions tools can make about them, which e.g. limits caching opportunities.

- _Macros_ &mdash; macros require multiple passes to expand, expand to often surprising amounts of hidden code, and impose limitations on partial parsing. Procudural macros have negative impacts similar to build scripts.

- _LLVM backend_ &mdash; LLVM produces good machine code, but runs relatively slowly.

- _Relying too much on the LLVM optimizer_ &mdash; Rust is well-known for generating a large quantity of LLVM IR and letting LLVM optimize it away. This is exacerbated by duplication from monomorphization.

- _Split compiler / package manager_ &mdash; although it is normal for languages to have a package manager seperate from the compiler, in Rust at least this results in both `cargo` and `rustc` having imperfect and redundant information about the overall compilation pipeline. As more parts of the pipeline are short-circuited for efficiency, more metadata needs to be transfered between instances of the compiler, mostly through the filesystem, which has overhead.

- _Per-compilation-unit code-generation_ &mdash; `rustc` generates machine code each time it compiles a crate, but it doesn't need to &mdash; with most Rust projects being statically linked, the machine code isn't needed until the final link step. The may be efficiencies to be had by completely separating analysis and code generation.

- _Single-threaded compiler_ &mdash; ideally, all CPUs are occupied for the entire compilation. This is not close to true with Rust today. And with the original compiler being single threaded, the language is not as friendly to parallel compilation as it might be. There is effort going into parallelizing the compiler, but it may never use all your cores.

- _Trait coherence_ &mdash; Rust's traits have a property called "coherence", which makes it impossible to define implementations that conflict with each other. Trait coherence imposes restrictions on where code is allowed to live such that it is difficult to decompose Rust abstractions into, small, easily-parallelizable compilation units.

- _Tests next to code_ &mdash; Rust encourages tests to reside in the same codebase as the code they are testing. With Rust's compilation model this requires compiling and linking that code twice, which is expensive, particularly for large crates.


## Recent work on Rust compile times

The situation isn't hopeless. Not at all. There is always work going on to improve Rust compile times, and there are still many avenues to be explored. I'm hopefull that we'll continue to see improvements. Here is a selection of the activity I'm aware of from the last year or two. Thanks to everybody who helps with this problem.

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

- [Cranelift backend][cbe]
  - Reduced debug compile times by used [cranelift] for code generation.
  - Developed by [@bjorn3]

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

[cranelift]: https://github.com/bytecodealliance/cranelift
[cbe]: https://www.reddit.com/r/rust/comments/enxgwh/cranelift_backend_for_rust/
[@bjorn3]: https://github.com/bjorn3
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

So Rust dug itself deep into a corner over the years and will probably be digging itself back out until the end of time (or the end of Rust &mdash; same thing, really &#x1F92A;). Can Rust compile-time be saved from Rust's own run-time success? Will TiKV ever build fast enough to satisfy my managers?

In the next episode, we'll deep-dive into the specifics of Rust's language design that cause it to compile slowly.

Stay Rusty, friends.


## Thanks

A number of people helped with this blog series. Thanks especially to Niko Matsakis, Graydon Hoare, and Ted Mielczarek for their insights, and Calvin Weng for proofreading and editing.


<!--
## Old notes

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

images:

- https://www.pixel4k.com/crab-4k-69365.html
- http://yesofcorsa.com/blue-crab/
- https://commons.wikimedia.org/wiki/File:K%C4%B1z%C4%B1l%C4%B1rmak_near_the_crab.jpg
- https://cdn4.gamepur.com/images/god-of-war/Mr_Krabs_And_Money.jpg
- http://nick.mtvnimages.com/nick/video/images/spongebob-squarepants/spongebob-squarepants-mr-krabs-formula-clip-16x9.jpg?quality=0.75&maxdimension=600&height=225&width=400
- https://www.bbc.co.uk/radioscotland/60s/moonlandings/05/


![taunting crab](https://brson.github.io/tmp/crab2.jpg)
![taunting crab](https://brson.github.io/tmp/crab3.jpg)
![taunting crab](https://brson.github.io/tmp/crab4.jpg)
![taunting crab](https://brson.github.io/tmp/crab5.jpg)

Some timings:

- TODO minutes - `rustboot` building Rust prior to being retired ([commit][cpre], [log][lpre])
- TODO minutes - `rustc` building Rust immidately after `rustboot`s retirement ([commit][cpost], [log][lpost])
- TODO minutes - `rustc` building Rust in 2020 ([commit][ctoday], [log][ltoday])

[cpre]: https://github.com/rust-lang/rust/commit/ef75860a0a72f79f97216f8aaa5b388d98da6480
[cpost]: https://github.com/rust-lang/rust/commit/6997adf76342b7a6fe03c4bc370ce5fc5082a869
[ctoday]: https://github.com/rust-lang/rust/commit/aa0769b92e60f5298f0b6326b8654c9b04351b98
[lpre]: todo
[lpost]: todo
[ltoday]: todo

bash -c "git log --oneline -1 && /usr/bin/time make 2>&1" | tee out.txt

bash -c "git log --oneline -1 && /usr/bin/time make ENABLE_OPTIMIZED=1 2>&1" | tee out.txt

llvm: https://gist.github.com/cb95b7b975b30fa1832174d0313db6d4

-->