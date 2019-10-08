# The Rust Compilation Model Calamity

## Or, The TiKV Compile-time Saga: Part 1

---

_The Rust programming language was designed for slow compilation times._

I assert this with high confidence because [I was in the room when the deed was done][it]. This whole Rust thing is all a [long con] the language designers played on you, the Rust user &mdash; adopt this crazy-fast, super-reliable language for your mission-critical products, and we'll slow your developers' productivity to a crawl.

<!-- TODO use the argument from authority above or no? -->

We totally got you.

![taunt]

[taunt]: https://brson.github.io/tmp/taunt.jpg

<!-- TODO image credit -->

[it]: todo
[long con]: todo

Am I joking? I don't know. What even is anything anymore, anyway?

I do know one thing though…


## Rust compile times suck

If you are reading this you probably know it too. We all know it. We mostly hate it.

I know it because nearly every Rust production user I talk to says it is _the thing_ they dislike the most about the language. I know it because so many of us indicated so [in the Rust 2019 survey][sur].

[sur]: todo

This is kinda infuriating, as almost everything that matters about Rust is pretty damn good. But Rust compile times are so, so bad.

Rust compile times are its biggest weakness.

At [PingCAP], we develop our distributed storage system, [TiKV], in Rust, and it compiles slow enough to discourage many in the company from using Rust. I recently spent some time ("some time" being months and months upon increasing months), along with several others on the TiKV team and its wider community, investigating TiKV's pitiable compile times.

[PingCAP]: todo
[TiKV]: todo

Over a series of posts I'll discuss what we have learned (maybe &mdash; maybe I'll just lazy and leave you wondering forever): why compiling Rust is slow, and/or feels slow; compile-time use cases; things we measured; things we want to measure but haven't or don't know how; ideas that improved compile times; ideas that did not improve compile times; how TiKV compile times have changed over time; suggestions for how to organize Rust projects that compile fast; and recent and future upstream improvements to compile times.

<!-- TODO: "maybe I'll just lazy" is hard to parse but I like it! cc @dcalvin -->

In this episode:

- [The Spectre of Poor Rust Compile Times at PingCAP](#user-content-the-spectre-of-poor-rust-compile-times-at-pingcap)
- [Preview: The TiKV Compile-time Saga so far](#user-content-preview-the-tikv-compile-time-saga-so-far)
- [Part 1: The Rust Compilation Model Calamity](#user-content-part-1-the-rust-compilation-model-calamity)
- [Bootstrapping Rust](#user-content-bootstrapping-rust)
- [(Un)virtuous cycles](#user-content-unvirtuous-cycles)
- [A brief aside about compile-time scenarios](#user-content-a-brief-aside-about-compile-time-scenarios)
- [Rust's designs for poor compile time](#user-content-rusts-designs-for-poor-compile-time)
- [Tradeoff #1: Monomorphized generics](#user-content-tradeoff-1-monomorphized-generics)
- [Aside: Generic forms in Rust](#user-content-generic-forms-in-rust)
- [Testing the impact of monomorphization on Rust compile times](#user-content-testing-the-impact-of-monomorphization-on-rust-compile-times)
- [Tradeoff #2: Huge compilation units](#user-content-tradeoff-2-huge-compilation-units)
- [Tradeoff #3: Trait coherence and the orphan rule](#user-content-tradeoff-3-trait-coherence-and-the-orphan-rule)
- [Tradeoff #4: LLVM](#user-content-tradeoff-4-llvm)
- [Tradeoff #5: Poor LLVM IR generation](#user-content-tradeoff-5-poor-llvm-ir-generation)
- [Tradeoff #6: Batch compilation](#user-content-tradeoff-6-batch-compilation)
- [Tradeoff #7: Build scripts](#user-content-tradeoff-7-build-scripts)
- [Tradeoff #8: Procedural macros](#user-content-tradeoff-8-procedural-macros)
- [Tradeoff #9: Fixed compilation profiles](#user-content-tradeoff-9-fixed-compilation-profiles)
- [All that stuff summarized](#user-content-all-that-stuff-summarized)
- [In the next episode of The TiKV Compile-time Saga](#user-content-in-the-next-episode-of-the-tikv-compile-time-saga)
- [Thanks](#user-content-thanks)


## The Spectre of Poor Rust Compile Times at PingCAP

At [PingCAP], my colleagues write [TiKV], the storage node of our distributed database, in Rust. They do this because they want this most important node in the system to be fast and reliable by construction, at least to the greatest extent reasonable.

It was mostly a great decision, and most people internally are mostly happy about it.

But many complain about how long it takes to build. For some a full rebuild might take ~15 minutes in development mode, and ~30 minutes in release mode. To developers of large systems projects this might not sound horrible, but it's much slower than what most developers are used to out of modern programming languages. TiKV is not even a particularly large system, with TODO total lines of Rust code. Building [Servo] or [Rust itself][r] is much, much more unpleasant.

[Servo]: todo
[r]: todo

Other nodes in the system are written in Go, which of course comes with a different set of advantages and disadvantages from Rust. The Go developers at PingCAP kinda hate and resent having to wait for the Rust components to build. They are used to a rapid build-test cycle.

Rust developers on the other hand are used to taking a lot of coffee breaks. (Or tea, or cigarettes, or sobbing, or whatever, as the case may be &mdash; Rust developers have the spare time to nurse their personal demons real good.)

_Internally at PingCAP, new code that would be appropriate to write in Rust is often written in Go, only because of the spectre of terrible compile times_.

<!-- TODO: can the above be backed up with an example if asked? -->

This is a company that is one of Rust's greatest advocates in Asia.


## Preview: The TiKV Compile-time Saga so far

The first entry in this series is just a story about why Rust compile times suck. Since it might take another entry or two to dive into concrete technical details of what we've done with TiKV's compile times, here's a pretty graph to capture your imagination, without comment.

![tikv-compile-timings]

[tikv-compile-timings]: https://brson.github.io/tmp/tikv-timings.svg


## Part 1: The Rust Compilation Model Calamity

Rust was designed for slow compilation times.

I mean, that wasn't _the goal_. As is often cautioned in debates among their designers, programming language design is full of tradeoffs. One of those fundamental tradeoffs is **run-time performance** vs. **compile-time performance**, and the Rust team nearly always (if not always) chose run-time over compile-time.

The intentional run-time / compile-time tradeoff isn't the only reason Rust compile times are horrific, but it's a big one. There are also language designs that are not crucial for run-time performance, but accidentally bad for compile time performance. The Rust compiler was also not designed to support fast compilation times.

So there are intrinsic language-design reasons, and accidental language-design reasons for Rust's bad compile times. Those mostly can't be fixed ever (though they may be mitigated by compiler improvements, design patterns, and language evolution). There are also accidental compiler-architecture reasons for Rust's bad compile times, which can generally be fixed through enormous engineering effort and time.

For context, if compilation time was not a core Rust design principle, what were Rust's core design principles? Here are a few:

- Practicality &mdash; it should be a language that can be and is used in the real world.
- Pragmatism &mdash; it should admit concessions to human usability and integration into systems as they exist, instead of attempting to maintain any kind of theoretical purity.
- Memory-safety &mdash; it must enforce memory safety, and not admit segmentation faults and other such memory-access violations.
- Performance &mdash; it must be in the same performance class as C and C++.
- Concurrency &mdash; it must provide modern solutions to writing concurrent code.

But it's not like we didn't put _any_ consideration into fast compile times. For example, for any analysis Rust needs to do, the team tried to ensure it would not suffer from exponential complexity (see e.g. [Swift's accidental exponential type-inference blow-up][sti]). Rust's design history though one of increasingly being sucked into the swamp of poor compile-time performance.

<!-- TODO: discuss the dangers of exponential complexity and how it happens? -->

Story time.

[sti]: todo


## Bootstrapping Rust

I don't remember when I realized that Rust's bad compile times were a strategic problem for the language, potentially a fatal mistake in the face of future Rust-alike competitors. For the first few years I wasn't too concerned, and I don't think most of my peers were either. I mostly remember that Rust compile time was always* bad, and like, whatever, that's just reality.

<!-- TODO dcalvin doesn't like "and like, whatever" -->

When I worked on Rust heavily it was common for me to have at least three copies of the repository on the computer, hacking on one while all the others were building and testing. I would start building workspace 1, switch screen/tmux tabs, remember what's going on over here in workspace 2, hack on that for a while, start building in workspace 2, switch tabs, etc. Little flow, constant context switching.

This was (and probably is) typical of other Rust developers too.

(*) OK, Rust's compile time actually wasn't _always_ terrible. The [first Rust compiler][frc], written in [OCaml], had extremely simple static analysis, and extremely naive code generation. It still builds today. Check it out if you're curious. I just did:

```
TODO
```

Only TODO minutes. Oh those halcyon days. For comparison here's a build of [today's compiler][tc] (as of TODO):

```
TODO
```

Lol. Just lol. The comparison is completely unfair, for reasons. But lol, right? If you are curious, the full build logs for both runs are [yonder].

<!-- TODO: maybe a small gif here? -->

[frc]: todo
[OCaml]: todo
[tc]: todo
[yonder]: todo

One of the immediate tasks upon Rust's initial open-sourcing in 2010 was to make it _self-hosting_ &mdash; that is, to rewrite the Rust compiler (`rustc`) in Rust. The Rust version of `rustc`, in addition to being written in Rust, would also use [LLVM] as its backend for generating machine code, instead of OCaml-`rustc`'s hand-written x86-specific code-generator.

Rust needed to become self-hosting as a means of "dog-fooding"

[LLVM]: todo

The first time Rust built itself was on TODO. [It took TODO hours][self-host].

[self-host]: todo

That first super-slow bootstrap was an anomaly of bad code-generation and other easily fixable early bugs (probably, I don't exactly recall). `rustc`'s performance quickly improved, and Graydon quickly [threw away the OCaml-`rustc`][nocaml] since there was nowhere near enough manpower and motivation to maintain parallel implementations. To compare with the previously-presented build times, that commit that drops OCaml bootstraps in TODO minutes under the same environment ([logs]).

[nocaml]: todo
[logs]: todo

This is where the long, gruelling history of Rust's tragic compile time began, TODO months after it was initially released in July 2010.

---

Note that it is not actually fair to compare Rust's own compile time to the compile time of Rust programs generally, but it is fun because the Rust build takes so hilariously long.

The reason for this is that Rust has to build itself 3 times to prove that it is self hosting. It does this in 3 "stages":

- _stage0_ - build from the previous release's binary. This proves that previous-Rust can build current-Rust.

- _stage1_ - build from current-Rust as built by previous-Rust. This proves that current-Rust can build current-Rust, but not that current-Rust (or next-Rust) built by current-Rust works (can build next-Rust).

- _stage2_ - build from current-Rust as built by current-Rust. This proves that current-Rust can build a working current-Rust (or next-Rust), and that the self-hosting cycle will continue.

<!-- todo verify stages -->

The details of how Rust's stages interact is fairly complex, but this explains the basic reason Rust has to be compiled three times. These three stages are necessarily serialized, which does nothing to help its own compile time.


## (Un)virtuous cycles

In the Rust project we like processes that reinforce and build upon themselves. This is one of the keys to Rust's success, both as a language and community.

As an obvious, hugely-successful example, consider [Servo]. Servo is a web browser built in Rust (technically it's still just a browser "engine"), and Rust was created with the explicit purpose of building Servo.

Rust and Servo are sister-projects. They were created by the same team (initially), at roughly the same time, and they evolved together. Not only was Rust built to create Servo, but Servo was built to inform the design of Rust.

The initial few years of both projects were extremely difficult, with both projects evolving in parallel. The metaphor of the [Ship of Theseus][st] comes to mind: we were constantly rebuilding the ship we were sailing, constantly rebuilding Rust in order to sail the Servo ship. There is no doubt that the experience of building Servo with Rust while simultaneously building Rust itself led directly to many of the good decisions that make Rust the practical language it is.

[st]: https://en.wikipedia.org/wiki/Ship_of_Theseus

As just one example of the Servo-Rust feedback loop, the decision to [drop green threading][gt] from Rust was heavily informed by the huge amount of FFI required to integrate SpiderMonkey into Servo, and green threading kills FFI performance. In fairness though, not everyone one the Servo team was thrilled about this Rust decision at the time, since it made writing massively-multithreaded code more challenging, though I hope they agree in retrospect that it was the right one.

[gt]: todo

The co-development of Rust and Servo created a [virtuous cycle] that allowed both projects to thrive. Today, Servo components are deeply integrated into Firefox, ensuring that Rust cannot die while Firefox lives.

Mission accomplished.

[virtuous cycle]: https://en.wikipedia.org/wiki/Virtuous_circle_and_vicious_circle

The previously-mentioned early self-hosting was similarly crucial to Rust's design, making Rust a superior language for building Rust compilers. Likewise, Rust and [WebAssembly] were developed in close collaboration, making WASM an excellent platform for running Rust, and Rust the first language beyond C and C++ with decent WASM support.

[WebAssembly]: todo

Sadly there was no such reinforcement to drive down compile times. The opposite is probably true: the more Rust became known as a "fast" language the more important it was to be "the fastest" language; and the more Rust's developers got used to developing their Rust projects across multiple branches, context switching between tasks as I described earlier, the less pressure was felt to address compile times.

That is, until Rust was actually released to production and met by a wide audince that was not used to such abhorrent compile times.

For years Rust [slowly boiled][boil] in its own poor compile times, not realizing how bad it had gotten until it was too late. It was 1.0. Those decisions were locked in. It was boiled.

[boil]: https://en.wikipedia.org/wiki/Boiling_frog

A lot of metaphores in this section! Sorry, I'm not usually like that.


## A brief aside about compile-time scenarios

It's tempting to talk about "compile-time improvements" broadly, without any further clarification, but there are many types of "compile-time", some that matter more or less to different people. The four main compile-time scenarios are:

- development profile / full rebuilds
- development profile / partial rebuilds
- release profile / full rebuilds
- release profile / partial rebuilds

The "development profile" entails compiler settings designed for fast compile times, slow run times, and maximum debuggability. The "release profile" entails compiler settings designed for fast run times, slow compile times, and, usually, minimum debuggability. In Rust, these are invoked with `cargo build` and `cargo build --release` respectively, and are the embodiment of the compile-time/run-time tradeoff.

A full rebuild is building the entire project from scratch, and a partial rebuild happens after modifying code in a previously built project. Partial rebuilds can notably benefit from [incremental compilation][ic].

[ic]: todo

In addition to those there are also

- test profile / full rebuilds
- test profile / partial rebuilds
- bench profile / full rebuilds
- bench profile / partial rebuilds

These are mostly similar to development mode and release mode respectively, though the interactions in cargo between development / test and release / bench can be subtle and surprising. There may be other profiles (TiKV has more), but those are the obvious ones for Rust, as built-in to cargo. Beyond that there are other scenarios, like typechecking only (`cargo check`), building just a single project (`cargo build -p`), single-core vs. multi-core, local vs. distributed, local vs. CI.

Compile time is also affected by human perception &mdash; it's possible for compile time to feel bad when it's actually decent, and to feel decent when it's actually not so great. This is one of the premises behind the [Rust Language Server][RLS] (RLS) &mdash; if developers are getting constant, real-time feedback about type errors in your IDE then it doesn't matter so much how long a full compile takes.

[RLS]: todo

So it's important to keep in mind through this series that there is a spectrum of semi-tunable possibilities from "fast compile/slow run" to "fast run/slow compile", there are different scenarios that affect compile time in different ways, and in which compile time affects perception in different ways, and that there are psychological factors at play.

It happens that for TiKV we've identified that the scenario we care most about with respect to compile time is "release profile / partial rebuilds". This also happens to be the scenario with the hardest tradeoffs and the least room for improvement. Yay. More about that in future installments.


## Rust's designs for poor compile time

The rest of this post details some of the biggest reasons that Rust's intrinsic design choices and historical architectural choices discourage fast compilation.


## Tradeoff #1: Monomorphized generics

Rust's approach to generics is the most obvious language feature to blame on bad compile times, and understanding how Rust translates generic functions to machine code is important to understanding the Rust compile-time/run-time tradeoff.

Generics generally are a complex topic, and Rust generics come in a number of forms that I'm not going to explore in-depth here. Rust has generic types, generic functions, and generic methods. For simplicity, we'll discuss generic functions, but they all are essentially the same (functions are effectively methods without a self-type, and generic types force all their methods to be generic).

As an example, consider the following `ToString` trait and the generic function `print`:

```rust
trait ToString {
    fn to_string(&self) -> String;
}

fn print<T: ToString>(v: T) {
     println!("{}", v.to_string());
}
```

This is not quite idiomatic Rust (you wouldn't use `ToString` like this), but close enough to be clear to a general audience: `print` will print to the console anything that can be converted to a `String` type. We say that "`print` is generic over type `T`, where `T` implements `ToString`". Thus I can call `print` with different types:

```rust
print("hello, world");
print(101);
```

The way a compiler translates these calls to `print` to machine code has a huge impact on both the compile-time and run-time characterics of the language.

When a generic function is called with a particular set of type parameters it is said to be _instantiated_ with those types.

Again, _in general_, for programming languages, there are two ways to translate a generic function:

1) translate the generic function for each set of instantiated type parameters, calling each trait method directly, but duplicating most of the generic function's machine instructions, or

2) translate the generic function just once, calling each trait method through a function pointer (via a ["vtable"]).

["vtable"]: todo

The first results in static method dispatch, the second in dynamic (or "virtual") method dispatch. The first is sometimes called "monomorphization", particularly in the context of C++ and Rust, a confusingly complex word for a simple idea. You can think of the word as meaning that the generic function representing multiple implementations (or forms) is translated such that each form is implemented by its own instance, each instance a single form. "Single form" &mdash; "mono morph". Ugh, I can't even explain it clearly. Horrible smarty-pants word.

Again to make this concrete, imagine a hand-translation of both the static dispatch and dynamic dispatch strategies for our two `print` instantiations. These are just illustrative examples, and not what the compiler actually produces.

First, the static dispatch:

```rust
fn print_str(v: &str) {
    println!("{}", v.to_string());
}

fn print_uint(v: u32) {
    println!("{}", v.to_string());
}
```

Then, the dynamic:

```rust
struct ToStringVTable {
    to_string: unsafe fn(self: *const Opaque) -> String,
}

unsafe fn print(v: *const Opaque, vtable: &ToStringVTable) {
    vtable.to_string(v);
}
```

In the first, there's no indirection, but two `print` functions; in the second there's a layer of indirection through function pointers in `ToStringVTable`, and only one `print` function. The second is ugly and can't be written safely &mdash; that's because `print` needs to pass an opaque pointer to the `self` argument` of the virtual method, which the method then unsafely casts to the correct type; when the compiler does this translation it always ensures that the type matches the vtable and the cast is safe to perform.

To make things real, let's also look at what the Rust compiler actually emits for these two cases.

First, a real Rust static dispatch test case:

```rust
trait ToString {
    fn to_string(&self) -> String;
}

fn print<T: ToString>(v: T) {
     println!("{}", v.to_string());
}

fn main() {
  print("hello, world");
  print(101);
}
```

And now, a real Rust dynamic dispatch test case:

```rust
trait ToString {
    fn to_string(&self) -> String;
}

fn print(v: &dyn ToString) {
     println!("{}", v.to_string());
}

fn main() {
  print("hello, world");
  print(101);
}
```

A selection of the assembly for the static case:

```
TODO
```

And the dynamic case:

```
TODO
```

OK, sure enough, we can see the same pattern: in the first, there's no indirection, but two `print` functions; in the second there's a layer of indirection through function pointers, and only one `print` function.

These two strategies represent a notoriously difficult tradeoff: the first creates lots of machine instruction duplication, forcing the compiler to spend time generating those instructions, and putting pressure on the instruction cache, but &mdash; crucially &mdash; dispatching all the trait method calls statically instead of through a function pointer. The second saves lots of machine instructions and takes work for the compiler to translate to machine code, but every trait method call is an indirect call through a function pointer, which is generally slower because the CPU can't know what instruction it is going jump to until the pointer is loaded.

It is often thought that the static dispatch strategy results in faster machine code, though as far as I know there is little, if any, research into the matter. Intuitively, it makes sense &mdash; if the CPU knows the address of all the functions it is calling it should be able to call them faster than if it has to first load the address of the function, then load the instruction code into the instruction cache. There are though a few factors that make this intuition suspect: first, modern CPUs have invested a lot of silicon into branch prediction, so once a function pointer has been used once it is likely to be called quickly the next time; second, monomorphization results in huge quantities of machine instructions, a phenomenon commonly referred to as "code bloat", which puts great pressure on the CPU's instruction cache.

C++ and Rust both strongly encourage monomorphization, both generate some of the fastest machine code of any programming language, and both have problems with code bloat. This seems to be evidence that the monomorphization strategy is indeed the faster of the two. There is though a curious counter-example: C. C has no generics at all, and C programs are often both the slimmest _and_ fastest in their class. Reproducing the monomorphization strategy in C requires using the ungainly C macro preprocessor, and modern object-orientation patterns in C are often vtable-based.

_The takeaway here is that it is a broadly thought by compiler engineers that monomorphiation results in somewhat faster generic code while taking considerably longer to compile._

Note that the monomorphization-compile-time problem is compounded in Rust because Rust translates generic functions in every crate (generally, "compilation unit") that instantiates them. That means that if, given our `print` example, crate `a` calls `print("hello, world")`, and crate `b` also calls `print("hello, world, or whatever")`, then both crate `a` and `b` will contain the monomorphized `print_str` function &mdash; the compiler does all the type-checking and translation work twice. It is thought that eliminating this duplication could reduce compile times by around TODO%. [See issue #TODO][mono-issue] for the current status.

[mono-issue]: todo

All that is only touching on the surface of the compile-time complexity introduced by monomorphization. I passed this draft by [Niko], the primary type theorist behind Rust, and he said, more-or-less

> so far, everything looks pretty accurate, except that I think the monomorphization area leaves out a lot of the complexity. It's definitely not just about virtual function calls.
>
> it's also things like foo.bar where the offset of bar depends on the type of foo
>
> many languages sidestep this problem by using pointers everywhere (including generic C, if you don't use macros)
>
> not to mention the construction of complex types like iterators, that are basically mini-programs fully instantiated and then customizable -- though this *can* be reproduced by a sufficiently smart compiler
>
> (in particular, virtual calls can be inlined too, though you get less optimization; I remember some discussion about this at the time, I think strcat was pointing out how virtual call inlining happens relatively late in the pipeline)

TODO: explain the `foo.bar` comment.

[Niko]: todo


## Aside: Generic forms in Rust

<!--
TODO worth doing this?
-->


## Testing the impact of monomorphization on Rust compile times

As I said before, I don't think there's much solid research into the actual impact of dynamic vs. static dispatch on compile times &mdash; just a lot of inherited wisdom (but leave comments if you have good links and I'll update this).

So let's do an experiment!

Obviously, this isn't going to be the definitive word on the subject, just a quick, Rust-specific experiment, but now I'm curious, and I want to give you some numbers to back this all up.

I want to measure the effect of increasing numbers of generic function instantiations on both compile-time and run-time. Due to the contrast between static call inlining vs. indirect call runtime overhead, I suspect there are multiple interesting cases, so I want to try both "deep" and "wide" instantiations, with deep codebases having a deep call graph with few generic type instantiations, and wide codebases a shallow call graph with many instantiations; add to that "in-between" cases with middling call graph depth and type instantiations.

So my experiment is going to be built around a code generator with two inputs: call-graph depth, generic type width. It will generate code like the following.

For static dispatch ("monomorphization"):

```rust
// depth = 2, width = 2

#![feature(test)]
extern crate test;

trait Io { fn do_io(&self); }

#[derive(Debug)]
struct T0(u8);
impl Io for T0 { fn do_io(&self) { black_box(self) } }

#[derive(Debug)]
struct T1(u8, u8);
impl Io for T0 { fn do_io(&self) { black_box(self) } }

fn lvl_0<T: Io>(v: T) { lvl_1(v) }
fn lvl_1<T: Io>(v: T) { v.do_io() }

fn main() {
    let v0 = T0(0);
	lvl_0(v0);
}
```

For dynamic dispatch:

```rust
// depth = 2, width = 2

#![feature(test)]
extern crate test;

trait Io { fn do_io(&self); }

#[derive(Debug)]
struct T0(u8);
impl Io for T0 { fn do_io(&self) { black_box(self) } }

#[derive(Debug)]
struct T1(u8, u8);
impl Io for T0 { fn do_io(&self) { black_box(self)} }

fn lvl_0(v: &dyn Io) { lvl_1(v) }
fn lvl_1(v: &dyn Io) { v.do_io() }

fn main() {
    let v0 = T0(0);
	lvl_0(v0);
}
```

The reason for "bottomming out" in an I/O function is to prevent the compiler from [optimizing away the entire program][optaway]. The downside of I/O is that it an I/O operation tends to be orders of magnitude slower than a compute operation, which makes it difficult to measure small changes in CPU usage. So our I/O is a call to [`black_box`], an assembly-language function that does nothing, but is opaque to the compiler, preventing it from optimizing away the entire program.

[optaway]: todo

TODO


## Tradeoff #2: Huge compilation units

A _compilation unit_ is the basic unit of work that a language's compiler operates on. In C and C++ the compilation unit is a source file. In Java it is a source file. In Rust the compilation unit is a _crate_, which is composed of many files.

- TODO impact of compilation unit on compile time
- TODO relation between compilation unit and backend codegen units
- TODO inlining tradeoffs
- TODO parallelism, incrementalism, codegen units

Rust crates don't have to be large, but there are a variety of factors that encourage them to be. The first is simply the relative complexity of adding new crates to a Rust project vs. adding a new module to a crate. New Rust projects tend to turn into monoliths unless given special attention to abstraction boundaries.

Within a crate, there are no fundamental restrictions on module interdependencies, though there are language features that allow some amount of information-hiding within a crate. The big advantage and risk of having modules in the same crate is that they can have _mutual dependencies_, two modules both depending on names exported from the other. Here's an example similar to many encountered in TiKV:

```rust
mod storage {
	use network::Message;

	pub struct Engine;

    impl Engine {
	    pub fn handle_message(&self, msg: Message) -> Result<()> { ... }
	}
}

mod network {
	use storage::Engine as StorageEngine;

	pub enum Message { ... }

	struct Server;

	impl Server {
		 fn handle_message(&self, msg: Message) -> Result<()> {
		 	 ...

			 self.engine.handle_message(msg)?;

			 ...
		 }
	}
}
```

Mutual dependencies are useful for reducing cognitive compleity by breaking up code, but as an abstraction boundary they are deceptive: they cannot be trivially reduced further into separate crates.

And that is because _dependencies between crates must form a [DAG]_ (a directed acyclic graph); they do not support mutual dependencies.

[DAG]: todo

Rust crates being daggish is mostly due to fundamental reasons of type checking and architectural complexity. If crates allowed for mutual dependencies then they would no longer be self-contained compilation units.

In preparation for this blog I asked a few people if they could recall the reasons why Rust crates must form a DAG, and Graydon gave a typically thorough and authoritative answer:

> TODO

Although driven by fundamental design constraints, the hard daggishness of crates should be consider a feature: it enforces careful abstractions, defines units of _parallel_ compilation, defines basically sensible codegen units, and dramaticaly reduces language and compiler complexity (even as the compiler likely moves toward "whole-program" compilation in the future).

Note the emphasis on _parallelism_ above. The crate DAG is the best source of compile-time parallelism we have access to. Cargo today will use the DAG to automatically divide work into parallel jobs TODO.

TODO mention 

So it's quite desirable for Rust code to be broken into crates that form a _wide_ DAG.

...

Probably the biggest reason that Rust crates tend to be large though is because of ...


## Tradeoff #3: Trait coherence and the orphan rule

Rust's trait system makes it difficult to make crates into abstraction boundaries because of a thing call the _orphan rule_.

Traits are the most common tool for creating abstractions in Rust. They are powerful, but like much of Rust's power, it comes with a tradeof.

The [orphan rule][or]


[or]: todo


## Tradeoff #4: LLVM

This is pretty simple and understandable. `rustc` uses LLVM to generate code. LLVM can generate very fast code, but it comes at a cost. LLVM is a very big system. In fact, LLVM code makes up the majority of the Rust codebase. And it doesn't operate particularly fast.

So even when `rustc` is generating debug builds, that are supposed to build fast, but are allowed to run slow, generating the machine code still takes a considerable amount of time.

In a TiKV release build, LLVM passes occupy TODO% of the build time, while in a debug build LLVM passes occupy TODO% of the build time.

LLVM being poor at generating slow code quickly is not in intrinsic property of the language though, and efforts are underway to create a second backend using [Cranelift], a code generator written in Rust, and designed for fast code generation.

[Cranelift]: todo

In addition to being integrated into `rustc` Cranelift is also being integrated into SpiderMonkey as its WebAssembly code generator.

It's not fair to blame all the code generation slowness on LLVM though. `rustc` isn't doing LLVM any favors by
the way it generates LLVM IR.


## Tradeoff #5: Poor LLVM IR generation

`rustc` is notorious for throwing huge gobs of unoptimized LLVM IR at LLVM and expecting it to optimize it all away. This is (probably) the main reason Rust debug binaries are so slow.

So LLVM is doing a lot of work to make Rust as fast as it is.

This is another problem with the compiler architecture &mdash; it was just easier to create Rust by leaning heavily on LLVM than to be clever about how much info `rustc` handed to LLVM to optimize.

So this can be fixed over time, and it's one of the main avenues the compiler team is pursuing to improve compile-time performance.

Remember earlier the discussion about how monomorphization works? How it duplicates function definitions for every combination of instantiated type parameters? Well, that's not only a source of machine code bloat but also LLVM IR bloat. Every one of those functions is filled with duplicated, unoptimized, LLVM IR.

`rustc` is slowly being modified to such that it can perform its own optimizations on its own MIR (mid-level IR), and crucially, the MIR representation is pre-monomorphization. That means that MIR-level optimizations only need to be done once per generic function, and in turn produce smaller monomorphized LLVM IR, that LLVM can (in theory) translate faster than it does with its unoptimized functions today.


## Tradeoff #6: Batch compilation

It turns out that the entire architecture of `rustc` is "wrong", and so is the architecture of most compilers ever written.

It is common wisdom that all compilers have an architecture like the following:

- the compiler consumes an entire compilation by parsing all of its source code into an AST
- through a succession of passes, that AST is refined into increasingly detailed data "intermediate representations" (IRs)
- the entire final IR is passed through a code generator to emit machine code

I am calling this the "batch compilation" model. Maybe that's what others call it. This architecture is vaguelly how compilers have been described academically for decades; it is how most compilers have historically been implemented; and that is how `rustc` was originally architected. But it is not an architecture that well-supports the workflows of modern developers and their IDEs, nor does it support fast recompilation.

Today, developers expect instant feedback about the code they are hacking. When they write a type error, the IDE should immediately put a red squiggle under their code and tell them about it. It should ideally do this even if the source code doesn't completely parse.

The batch compilation model is poorly suited for this. It requires that entire compilation units be re-analyzed for every incremental change to the source code, in order to produce incremental changes to the analysis. In the last decade or so, the thinking among compiler engineers about how to construct compilers has been shifting from batch compilation to "responsive compilation", by which the compiler can run the entire compilation pipeline on the smallest subset of code possible to get a particular answer desired, as quickly as possible. For example, with responsive compilation one can ask "does this function type check?", or "what are the type dependencies of this structure?".

This ability lends itself to the _perception_ of compiler speed, since the user is constantly getting necessary feedback while they work. It can dramatically shorten the feedback cycle for correcting type checking errors, and in Rust, getting the program to successfully type check takes up a huge proportion of developers' time.

I'm sure the prior art is extensive, but a notable modern responsive compiler is the [Roselyn] .NET compiler; and the concept of responive compilers has recently been advanced significantly with the adoption of the [Language Server Protocol][lsp]. Both are Microsoft projects.

In Rust today we support this IDE use case with the [Rust Language Server][RLS] (the RLS). But many Rust developers will know that the RLS experience can be pretty disappointing, with huge latency between typing and getting feedback. Sometimes the RLS fails to find the expected results, or simply fails compeletely. The failures of the RLS are almost entirely due to being built on top of a batch-model compiler, and not a responsive compiler.

Taken to the limit, a responsive compiler architecture naturally lends itself to quickly responding to requests like "regenerate machine code, but only for functions that are changed since the last time the compiler was run". So not only does responsive compilation support the IDE analysis use case, but also the recompile-to-machine-code use case. Today, this second use case is supported in `rustc` with "incremental compilation", but it is fairly crude, with a great deal of duplicated work on every compiler invocation. We should expect that, as `rustc` becomes more responsive, incremental compilation will ultimately do the minimal work possible to recompile only what must be recompiled.

There are though tradeoffs in the quality of machine code generated via incremental compilation &mdash; due to the mysterious challenges of inlining, incrementally-recompiled code is unlikely to ever be as fast as highly-optimized, batch-compiled machine code. In other words, you probably won't ever want to use incremental compilation for your production releases, but it can drastically speed up the development experience, while producing relatively fast code.

Niko spoke about this architecture in his ["Responsive compilers" talk at PLISS 2019][resp]. It is an entirely watchable talk about compiler engineering and I recommend checking it out.

[resp]: https://www.youtube.com/watch?v=N6b44kMS6OM
[RLS]: todo
[Roselyn]: https://en.wikipedia.org/wiki/.NET_Compiler_Platform
[lsp]: todo

In that talk he also provided some examples of how the Rust language was accidentally designed poorly for responsive compilation, which I think is quite illustrative.

TODO


## Tradeoff #7: Build scripts

## Tradeoff #8: Procedural macros

## Tradeoff #9: Fixed compilation profiles


## All that stuff summarized




## In the next episode of The TiKV Compile-time Saga

Things are looking dire for Rust developers' productivity, and TiKV hackers are grumbling during their frequent coffee breaks! Can Rust succeed? Can Rust compile TiKV fast enough to prevent PingCAP's product managers from &nbsp; (╯°□°）╯︵&nbsp;┻━┻ &nbsp; and rewriting the entire thing in C++ or Go or Pony?

In the next entry of this series we'll talk about the nuances of the many Rust compile time use cases, which aspects of Rust affect which use cases, how users percieve and are affected by Rust compile time, and all the ideas the TiKV team came up with to free ourselves from this Rusting, suffering life.

Stay Rusty, friends.


## Thanks

This post has been in draft for a long time, and has benefited from the input and review of several people. Thanks especially to Calvin Weng for the reviews. Thanks to Niko Matsakis for advice about Rust's compile time behavior. Thanks to Graydon Hoare for recollections about Rust's design. Thanks to others for their patience about my extreme procrastination.


<!--
### TODO

- https://raphlinus.github.io/rust/2019/08/21/rust-bloat.html
- static linking?
-->

<!--

## Niko conversation

Niko Matsakis, [19.09.19 17:19]
so far, everything looks pretty accurate, except that I think the monomorphization area leaves out a lot of the complexity. It's definitely not just about virtual function calls.

Niko Matsakis, [19.09.19 17:19]
it's also things like foo.bar where the offset of bar depends on the type of foo

Niko Matsakis, [19.09.19 17:19]
many languages sidestep this problem by using pointers everywhere (including generic C, if you don't use macros)

Niko Matsakis, [19.09.19 17:20]
not to mention the construction of complex types like iterators, that are basically mini-programs fully instantiated and then customizable -- though this *can* be reproduced by a sufficiently smart compiler

Niko Matsakis, [19.09.19 17:21]
(in particular, virtual calls can be inlined too, though you get less optimization; I remember some discussion about this at the time, I think strcat was pointing out how virtual call inlining happens relatively late in the pipeline)

Niko Matsakis, [19.09.19 17:21]
anyway it doesnt' change your basic point

Niko Matsakis, [19.09.19 17:22]
[In reply to Brian Anderson]
Hmm. My experience has often been that this is hard in all languages. =) But in (e.g.) Java when I would do it, I often wound up doing a lot of downcasting in the "concrete" impls

Niko Matsakis, [19.09.19 17:23]
that is, I'd have a bunch of interface types that referenced one another and some class types that implement them, and the class types would depend on private details of one another, so you'd have to downcast

Niko Matsakis, [19.09.19 17:23]
(that too relies on everything being a pointer)

Niko Matsakis, [19.09.19 17:24]
the orphan rule is part of it but I don't feel like it's *the* thing

Brian Anderson, [05.10.19 17:06]
[In reply to Niko Matsakis]
does this only come in to play with associated types?

Niko Matsakis, [05.10.19 20:06]
no

Niko Matsakis, [05.10.19 20:06]
struct Foo<T> { x: u32, y: T, z: f64 }

Niko Matsakis, [05.10.19 20:06]
fn bar<T>(f: Foo<T>) -> f64 { f.z }

Niko Matsakis, [05.10.19 20:07]
as I recall, before we moved to monomorphization, we had to have two paths for everything: the easy, static path, where all types were known to LLVM, and the horrible, dynamic path, where we had to generate the code to dynamically compute the offsets of fields and things

Niko Matsakis, [05.10.19 20:07]
unsurprisingly, the two were only rarely in sync

Niko Matsakis, [05.10.19 20:07]
which was a common source of bugs

Niko Matsakis, [05.10.19 20:07]
I think a lot of this could be better handled today -- we have e.g. a reasonably reliable bit of code that computes Layout, we have MIR which is a much simpler target -- so I am not as terrified of having to have those two paths

Niko Matsakis, [05.10.19 20:08]
but it'd still be a lot of work to make it all work

Niko Matsakis, [05.10.19 20:08]
there was also stuff like the need to synthesize type descriptors on the fly (though maybe we could always get by with stack allocation for that)

Niko Matsakis, [05.10.19 20:08]
e.g., fn foo<T>() { bar::<Vec<T>>(); } fn bar<U>() { .. }

Niko Matsakis, [05.10.19 20:09]
here, you have a type descriptor for T that was given to you dynamically, but you have to build the type descriptor for Vec<T>

Niko Matsakis, [05.10.19 20:09]
and then we can make it even worse

Niko Matsakis, [05.10.19 20:09]
fn foo<T>() { bar::<Vec<T>>(); } fn bar<U: Debug>() { .. }

Niko Matsakis, [05.10.19 20:09]
now we have to reify all the IMPLS of Debug

Niko Matsakis, [05.10.19 20:09]
so that we can do trait matching at runtime

Niko Matsakis, [05.10.19 20:09]
because we have to be able to figure out Vec<T>: Debug, and all we know is T: Debug

Niko Matsakis, [05.10.19 20:10]
we might be able to handle that by bubbling up the Vec<T> to our callers...

-->

<!--

## Graydon conversation

brson: Do you remember why rust crates must form a dag? Niko doesn't seem to remember the original reason

graydon: Couple reasons

graydon: Prohibits mutual recursion between definitions across crates, allowing both an obvious deterministic bottom-up build schedule without needing to do some fixpoint iteration or separate declarations from definitions
and enables phases that need to traverse a complete definition, like typechecking, to happen crate-at-a-time (enabling _some_ degree of incrementality / parallelism).

graydon: (once you know you've seen all the cycles in a recursive type you can check it for finiteness and then stop expanding it at any boxed variants -- even if they cross crates -- and put a lazy / placeholder definition in those boxed edges; but you need to know those variants don't cycle back!)

graydon: (I do not know if rustc does anything like this anymore)

graydon: cyclicality concerns are even worse with higher order modules, which I was spending quite a lot of time studying when working on the early design. most systems I've seen require you to paint some kind of a boundary around a group of mutually-recursive definitions to be able to resolve them, so the crate seemed like a natural unit for that.

graydon: then there is also the issue of versioning, which I think was pretty heavy in my mind (especially after the experience with monotone and git): a lot of versioning questions don't really make sense without acyclic-by-construction references. Like if A 1.0 depends on B 1.0 which depends on A 2.0 you, again, need to do something weird and fixpointy and potentially quite arbitrary and hard to explain to anyone in order to resolve those dependencies.

graydon: also recall we wanted to be able to do hot code loading early on, which means that much like version-resolving, compiling or linking, your really in a much simpler place if there's a natural topological order to which things you have to load or unload. you can decide whether a crate is still live just by reference-counting, no need to go figure out cyclical dependencies and break them in some random order, etc.

graydon: I'm not sure which if any of these was the dominant concern. If I had to guess I'd say avoiding problems with separate compilation of recursive definitions and managing code versioning. recall the language in the manual / the rationale for crates: "units of compilation and versioning". Those were the consideration for their existence as separate from modules. Modules get to be recursive. Crates, no. Because of things to do with "compilation and versioning".

brson: I'm building a list of ways that rusts design discourages fast compilation

brson: The crate dag is something in thinking about, but I think it's really trait coherence that makes crate decomposition so hard

brson: Hard to break down rusts huge compilation units, and even when you do, more crates create new compile time tradeoffs

graydon: Yeah. I think there's a pretty long list. The original design compiled pretty fast but we kept making choices that traded off compile time for something else

graydon: (I don't mean to be some asshole who pretends the original design got it all right -- I know it was too slow and the choices we made along the way were all well-justified on their own -- just that some of the design choices that were compile-speed-focused wound up not really mattering given later choices. like you can't do nearly as much per-crate stuff as initially. cross-crate inlining and monomorphization really makes the whole crate model a little dubious. and it's not like we do hot code loading or even ABI-safe relinking without total recompilation now anyways.)

graydon: (was too slow => the runtime perf of the dynamic-polymorphic code was too slow. compile time was fast! haha)

graydon: honestly I'm not sure if there's a better answer in the systems niche aside from something like what rust is moving towards: a big soup of definitions with a many-layered stack of memoizing queries on top.

graydon: runtime perf is something nobody is willing to trade-away even the slightest bit of. so anything that could act as a "compiler firewall" (as C++ people call boxing / uniform representation idioms) is mostly rejected on principle.

graydon: I think modern c patterns make different tradeoffs - much more dynamic dispatch in c than c++/rust and nobody's complaining

graydon: to an extent I agree. it's tricky to get just right. that was certainly the argument I made to myself when initially designing the generics system! it's even there in the notes: "kernel people use virtual dispatch for everything and seem not to mind"

brson: I'd like to see good numbers on the tradeoffs involved in all the monomorphization code duplication

brson: I doubt anybody had studied it deeply

graydon: but they also use fewer layers of abstraction and a fair number of inline functions and macros. maybe just giving them the toolset with very clear control over the phase distinction is best.

graydon: I got the impression watching objc programmers that simply having a clear cost model in the "C parts" and the "dynamic OO messaging parts" meant they could balance the thing themselves pretty well.

graydon: yeah I agree I think it's only lightly studied

graydon: I was hoping boxed object types in rust would also get more love, but it's not _just_ representation, it's also type erasure (bounded universal vs. existential) which winds up not really substitutable at the API level.

graydon: Having better control over static vs dynamic dispatch is probably my biggest wish for rust++

graydon: We tried in rust but the balance is heavy for monomorphization

graydon: swift doesn't give you _control_ over it but it does have a dial that the compiler and its optimizers freely turns between the different points on the spectrum

graydon: yeah. and in rust that had significant downstream implications like "we generate bad LLVM code to begin with, so now we generate WAY TOO MUCH of that bad code". and like the duplicate-instance-elimination stuff .. I'm not sure if it ever got done or paid off, I remember you were working on it but that seems costly too.

graydon: *shrug* I mean I don't think rust did too bad. there are obvious places to go back and try again but it planted a pretty good stake in the ground :)

graydon: (when I use it, honestly the thing that pisses me off the most is not the compile times, though they are annoying; it's the trait tetris. I waste hours and hours trying to figure out the right factoring to work with the grain of the rules, make a set of traits that "works right" together and with other libs. invariably seem to fall back on macros.)

graydon: Niko has estimates about how much work the generic dedupe would save, and it's more modest than I would expect

graydon: mhm. I expect also there's a degree to which generic de-duplication interacts poorly with the "no indirection" idiom: everything's representationally-specialized to its arguments in such a way that it can't be reused with other arguments.

graydon: LLVM gained an interesting pass recently called "outlining", did you see it?

graydon: might not help much with compile times -- it's quite late in the passes for rust -- but it might make binaries smaller. and maybe if you spend all your time in DAGIsel it'd cut down on codegen, idk.

graydon: https://www.youtube.com/watch?v=naF9r8O_3aY

graydon: eh, it's not a game-changing size win anyway

graydon: idk lately I've got interested in array languages which have such a different code pattern they're almost incomparable. everything is SoA to an extent that you literally precompile the loops into the runtime once for each combination of scalar types, and then they're always 100% reused. super weird world.

-->