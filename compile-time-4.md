# Trait coherence -- a compile-time nuisance

![header image](https://brson.github.io/tmp/calamity-header.jpg)

The Rust programming language compiles fast software slowly.

In this series we explore Rust's compile times within the context of [TiKV], the key-value store
behind the [TiDB] database.

[TiKV]: https://github.com/tikv/tikv
[TiDB]: https://github.com/pingcap/tidb

&nbsp;


## Rust Compile-time Adventures with TiKV: Episode 4

In [the previous post in the series][prev] we covered Rust's early development history, and how it led to a series of decisions that resulted in a high-performance language that compiles slowly. This time we'll go into detail about some of the reasons that Rust's design choices discourage fast compilation.

[prev]: https://pingcap.com/blog/rust-compilation-model-calamity/


- [Tradeoff #3: Trait coherence and the orphan rule](#user-content-tradeoff-3-trait-coherence-and-the-orphan-rule)
- [Tradeoff #4: LLVM and poor LLVM IR generation](#user-content-tradeoff-4-llvm-and-poor-llvm-ir-generation)
- [Tradeoff #5: Batch compilation](#user-content-tradeoff-5-batch-compilation)
- [Tradeoff #6: Build scripts and procedural macros](#user-content-tradeoff-6-build-scripts-and-procedural-macros)
- [Tradeoff #7: Static linking](#user-content-tradeoff-7-static-linking)
- [All that stuff summarized](#user-content-all-that-stuff-summarized)
- [In the next episode of Rust Compile-time Adventures with TiKV](#user-content-in-the-next-episode-of-rust-compile-time-adventures-with-tikv)
- [Thanks](#user-content-thanks)


## Tradeoff #2: Huge compilation units

A _compilation unit_ is the basic unit of work that a language's compiler operates on. In C and C++ the compilation unit is a source file. In Java it is a source file. In Rust the compilation unit is a _crate_, which is composed of many files.

The size of compilation units incur a number of tradeoffs. Larger compilation units take longer to analyze, translate, and optimize than smaller crates. And in general, when a change is made to a single compilation unit, the whole compilation unit must be recompiled.

More, smaller crates improve the _perception_ of compile time, if not the total compile time, because a single change may force less of the project to be recompiled. This benefits the "partial recompilation" use cases. A project with more crates though may do more work on a full recompile due to a variety of factors, which I will summarize at the end of this section.

Rust crates don't have to be large, but there are a variety of factors that encourage them to be. The first is simply the relative complexity of adding new crates to a Rust project vs. adding a new module to a crate. New Rust projects tend to turn into monoliths unless given special attention to abstraction boundaries.


### Dependency graphs and unstirring spaghetti

Within a crate, there are no fundamental restrictions on module interdependencies, though there are language features that allow some amount of information-hiding within a crate. The big advantage and risk of having modules coexist in the same crate is that they can be _mutual dependent_, two modules both depending on names exported from the other. Here's an example similar to many encountered in TiKV:

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

Modules with mutual dependencies are useful for reducing cognitive complexity simply because they break up code into smaller units. As an abstraction boundary though they are deceptive: they are not truly independent, and cannot be trivially reduced further into separate crates.

And that is because _dependencies between crates must form a directed acyclic graph (a DAG)_; they do not support mutual dependencies.

Rust crates being daggish is mostly due to fundamental reasons of type checking and architectural complexity. If crates allowed for mutual dependencies then they would no longer be self-contained compilation units.

In preparation for this blog I asked a few people if they could recall the reasons why Rust crates must form a DAG, and [Graydon] gave a typically thorough and authoritative answer:

[Graydon]: github.com/graydon/

> graydon: Prohibits mutual recursion between definitions across crates, allowing both an obvious deterministic bottom-up build schedule without needing to do some fixpoint iteration or separate declarations from definitions
and enables phases that need to traverse a complete definition, like typechecking, to happen crate-at-a-time (enabling _some_ degree of incrementality / parallelism).

> graydon: (once you know you've seen all the cycles in a recursive type you can check it for finiteness and then stop expanding it at any boxed variants -- even if they cross crates -- and put a lazy / placeholder definition in those boxed edges; but you need to know those variants don't cycle back!)

> graydon: (I do not know if rustc does anything like this anymore)

> graydon: cyclicality concerns are even worse with higher order modules, which I was spending quite a lot of time studying when working on the early design. most systems I've seen require you to paint some kind of a boundary around a group of mutually-recursive definitions to be able to resolve them, so the crate seemed like a natural unit for that.

> graydon: then there is also the issue of versioning, which I think was pretty heavy in my mind (especially after the experience with monotone and git): a lot of versioning questions don't really make sense without acyclic-by-construction references. Like if A 1.0 depends on B 1.0 which depends on A 2.0 you, again, need to do something weird and fixpointy and potentially quite arbitrary and hard to explain to anyone in order to resolve those dependencies.

> graydon: also recall we wanted to be able to do hot code loading early on, which means that much like version-resolving, compiling or linking, your really in a much simpler place if there's a natural topological order to which things you have to load or unload. you can decide whether a crate is still live just by reference-counting, no need to go figure out cyclical dependencies and break them in some random order, etc.

> graydon: I'm not sure which if any of these was the dominant concern. If I had to guess I'd say avoiding problems with separate compilation of recursive definitions and managing code versioning. recall the language in the manual / the rationale for crates: "units of compilation and versioning". Those were the consideration for their existence as separate from modules. Modules get to be recursive. Crates, no. Because of things to do with "compilation and versioning".

> graydon: I cannot make a simple argument about this because I'm still not smart enough about module systems — the full thing is laid out in [dreyer's thesis] and discussed in shorter [slide-deck form here][sdfh] — but suffice to say that recursive modules make it possible to see the "same" opaque type through two paths that should probably be considered equal but aren't easily determined to be so, I think in part due to the mix of opacity that modules provide and the fact that you have to partly look through that opacity to resolve recursion. so anyway I decided this was probably getting into "research" and I should just avoid the problem space, go with acyclic modules.

[dreyer's thesis]: https://www.cs.cmu.edu/~rwh/theses/dreyer.pdf
[sdfh]: http://macqueenfest.cs.uchicago.edu/slides/dreyer.pdf

Although driven by fundamental constraints, the hard daggishness of crates is useful for a number of reasons: it enforces careful abstractions, defines units of _parallel_ compilation, defines basically sensible codegen units, and dramaticaly reduces language and compiler complexity (even as the compiler likely moves toward "whole-program" compilation in the future).

Note the emphasis on _parallelism_. The crate DAG is the simplest source of compile-time parallelism Rust has access to. Cargo today will use the DAG to automatically divide work into parallel compilation jobs.

So it's quite desirable for Rust code to be broken into crates that form a _wide_ DAG.

In my experience though projects tend to start in a single crate, without great attention to their internal dependency graph, and once compilation time becomes an issue, they have already created a spaghetti dependency graph that is difficult to refactor into smaller crates.
It happened to Servo, and it has also been my experience on TiKV, where I have made multiple aborted attempts to extract various modules from the main program, in long sequences of commits that untangle internal dependencies. I suspect that avoiding problematic monoliths is something
that Rust devs learn with experience, but it is a repeating phenomenon in large Rust projects.


### Internal parallelism

So Rust crates tend to be large, and even when the crate DAG is complex, there are almost always bottlenecks where there is only one compiler instance running, working on a single crate.

So in addition to `cargo`s parallel crate compilation, `rustc` itself is parallel over a single crate. It wasn't designed to be parallel though, so its parallelism is limited and hard-won.

Today the only real internal parallelism in `rustc` is the use of [_codegen units_], by which `rustc` automatically divides a crate into multiple LLVM modules during translation. By doing this it can perform code generation in parallel.

And combined with [_incremental compilation_], it can avoid re-translating codegen units which have not changed from run to run, decreasing partial rebuild time. Unfortunately, the impact of codegen units and incremental compilation on both compile-time and run-time performance is hard to predict: improving rebuild time depends on `rustc` successfully dividing a crate into independent units that are unlikely to force each other to recompile when changed, and its not obvious how humans should write their code to help `rustc` in this task; and arbitrarily dividing up a crate into codegen units creates arbitrary barriers to inlining, causing unexpected de-optimizations.

[_codegen units_]: https://doc.rust-lang.org/rustc/codegen-options/index.html
[_incremental compilation_]: https://rust-lang.github.io/rustc-guide/queries/incremental-compilation.html

The rest of the compiler's work is completely serial, though soon it should [perform some analysis in parallel][parc].

[parc]: https://internals.rust-lang.org/t/help-test-parallel-rustc/11503/14


### Large vs. small crates

The number of factors affected by compilation unit size is large, and I've given up trying to explain them all coherently. Here's a list of some of them.

- Compilation unit parallelism &mdash; as discussed, parallelising compilation units is trivial.

- Inlining and optimization &mdash; inlining happens at the compilation unit level, and inlining is the key to unlocking optimization, so larger compilation units are better optimized. This story is complicated though by link-time-optimization (LTO).

- Optimization complexity &mdash; optimization tends to have superlinear complexity in code size, so biger compilation units increase compilation time non-linearly.

- Downstream monomorphization &mdash; generics are only translated once they are instantiated, so even with smaller crates, their generic types will not be translated until later. This can result in the "final" crate having a disproportionate amount of translation compared to the others.

- Generic duplication &mdash; generics are translated in the crate which instantiates them, so more crates that use the same generics means more translation time.

- Link-time optimization (LTO) &mdash; release builds tend to have a final "link-time optimization" step that performs optimizations across multiple code units, and it is extremely expensive.

- Saving and restoring metadata &mdash; Rust needs to save and load metadata about each crate and each dependency, each time it is run, so more crates means more redundant loading.

- Parallel "codegen units" &mdash; `rustc` can automatically split its LLVM IR into multiple compilation units, called "codegen units". The degree to which it is effective at this depends a lot on how a crate's internal dependencies are organized and the compilers ability to understand them. This can result in faster partial recompilation, at the expense of optimization, since inlining opportunities are lost.

- Compiler-internal parallelism &mdash; Parts of `rustc` itself are parallel. That internal parallelism has its own unpredictable bottlenecks and unpredictable interactions with external build-system parallelism.

Unfortunately, because of all these variables, it's not at all obvious for any given project what the impact of refactoring into smaller crates is going to be. Anticipated wins due to increased parallelism are often erased by other factors such as downstream monomorphization, generic duplication, and LTO.


## Tradeoff #3: Trait coherence and the orphan rule

Rust's trait system makes it challenging to use crates as abstraction boundaries because of a thing call the _orphan rule_.

Traits are the most common tool for creating abstractions in Rust. They are powerful, but like much of Rust's power, it comes with a tradeoff.

The [orphan rule][or] helps maintain [trait coherence], and exists to ensure that the Rust compiler never encounters two implementations of a trait for the same type. If it were to encounter two such implementations then it would need to resolve the conflict while ensuring that the result is sound.

What the orphan rule says, essentially, is that for any `impl`, either the _trait_ must be defined in the current crate, or the _type_ must be defined in the current crate.

This can create a tight coupling between abstractions in Rust, discouraging decomposition into crates &mdash; sometimes the amount of ceremony, boilerplate and creativity it takes to obey Rust's coherence rules, while also maintaining principled abstraction boundaries, doesn't feel worth the effort, so it doesn't happen.

This results in large crates which increase partial rebuild time.

There's subject deserves more examples and consideration, but I haven't the time for it now.

Haskell's type classes, on which Rust's traits are based, do not have an orphan rule. At the time of Rust's design, this was thought to be problematic enough to correct.

[or]: https://smallcultfollowing.com/babysteps/blog/2015/01/14/little-orphan-impls/
[trait coherence]: https://doc.rust-lang.org/reference/items/implementations.html#trait-implementation-coherence


## Tradeoff #4: LLVM and poor LLVM IR generation

`rustc` uses LLVM to generate code. LLVM can generate very fast code, but it comes at a cost. LLVM is a very big system. In fact, LLVM code makes up the majority of the Rust codebase. And it doesn't operate particularly fast.

So even when `rustc` is generating debug builds, that are supposed to build fast, but are allowed to run slow, generating the machine code still takes a considerable amount of time.

In a TiKV release build, LLVM passes occupy 84% of the build time, while in a debug build LLVM passes occupy 35% of the build time ([full details gist][tpg]).

[tpg]: https://gist.github.com/brson/ba22165d6da4a976b278b0896db7e4e4

LLVM being poor at quickly generating code (even if the resulting code is slow) is not in intrinsic property of the language though, and efforts are underway to create a second backend using [Cranelift], a code generator written in Rust, and designed for fast code generation.

[Cranelift]: https://github.com/bytecodealliance/cranelift

In addition to being integrated into `rustc` Cranelift is also being integrated into SpiderMonkey as its WebAssembly code generator.

It's not fair to blame all the code generation slowness on LLVM though. `rustc` isn't doing LLVM any favors by
the way it generates LLVM IR.

`rustc` is notorious for throwing huge gobs of unoptimized LLVM IR at LLVM and expecting LLVM to optimize it all away. This is (probably) the main reason Rust debug binaries are so slow.

So LLVM is doing a lot of work to make Rust as fast as it is.

This is another problem with the compiler architecture &mdash; it was just easier to create Rust by leaning heavily on LLVM than to be clever about how much info `rustc` handed to LLVM to optimize.

So this can be fixed over time, and it's one of the main avenues the compiler team is pursuing to improve compile-time performance.

Remember earlier the discussion about how monomorphization works? How it duplicates function definitions for every combination of instantiated type parameters? Well, that's not only a source of machine code bloat but also LLVM IR bloat. Every one of those functions is filled with duplicated, unoptimized, LLVM IR.

`rustc` is slowly being modified to such that it can perform its own optimizations on its own MIR (mid-level IR), and crucially, the MIR representation is pre-monomorphization. That means that MIR-level optimizations only need to be done once per generic function, and in turn produce smaller monomorphized LLVM IR, that LLVM can (in theory) translate faster than it does with its unoptimized functions today.


## Tradeoff #5: Batch compilation

It turns out that the entire architecture of `rustc` is "wrong", and so is the architecture of most compilers ever written.

It is common wisdom that all compilers have an architecture like the following:

- the compiler consumes an entire compilation unit by parsing all of its source code into an AST
- through a succession of passes, that AST is refined into increasingly detailed "intermediate representations" (IRs)
- the entire final IR is passed through a code generator to emit machine code

This is the batch compilation model. This architecture is vaguelly how compilers have been described academically for decades; it is how most compilers have historically been implemented; and that is how `rustc` was originally architected. But it is not an architecture that well-supports the workflows of modern developers and their tooling, nor does it support fast recompilation.

Today, developers expect instant feedback about the code they are hacking. When they write a type error, the IDE should immediately put a red squiggle under their code and tell them about it. It should ideally do this even if the source code doesn't completely parse.

The batch compilation model is poorly suited for this. It requires that entire compilation units be re-analyzed for every incremental change to the source code, in order to produce incremental changes to the analysis. In the last decade or so, the thinking among compiler engineers about how to construct compilers has been shifting from batch compilation to "responsive compilation", by which the compiler can run the entire compilation pipeline on the smallest subset of code possible to answer a particular question, as quickly as possible. For example, with responsive compilation one can ask "does this function type check?", or "what are the type dependencies of this structure?".

This ability lends itself to the _perception_ of compiler speed, since the user is constantly getting necessary feedback while they work. It can dramatically shorten the feedback cycle for correcting type checking errors, and in Rust, getting the program to successfully type check takes up a huge proportion of developers' time.

I imagine the prior art is extensive, but a notable innovative responsive compiler is the [Roselyn] .NET compiler; and the concept of responsive compilers has recently been advanced significantly with the adoption of the [Language Server Protocol][lsp]. Both are Microsoft projects.

In Rust today we support this IDE use case with the [Rust Language Server][RLS] (the RLS). But many Rust developers will know that the RLS experience can be pretty disappointing, with huge latency between typing and getting feedback. Sometimes the RLS fails to find the expected results, or simply fails compeletely. The failures of the RLS are mostly due to being built on top of a batch-model compiler, and not a responsive compiler.

The RLS is gradually being supplanted by [`rust-analyzer`], which amounts to essentially a ground-up rewrite of `rustc`, at least through its analysis phases, to support responsive compilation. It's expected that over time [`rust-analyzer` and `rustc` will share increasing amounts of code][libraryification].

[`rust-analyzer`]: https://github.com/rust-analyzer/rust-analyzer
[libraryification]: http://smallcultfollowing.com/babysteps/blog/2020/04/09/libraryification/

Taken to the limit, a responsive compiler architecture naturally lends itself to quickly responding to requests like "regenerate machine code, but only for functions that are changed since the last time the compiler was run". So not only does responsive compilation support the IDE analysis use case, but also the recompile-to-machine-code use case. Today, this second use case is supported in `rustc` with "incremental compilation", but it is fairly crude, with a great deal of duplicated work on every compiler invocation. We should expect that, as `rustc` becomes more responsive, incremental compilation will ultimately do the minimal work possible to recompile only what must be recompiled.

There are though tradeoffs in the quality of machine code generated via incremental compilation &mdash; due to the mysterious challenges of inlining, incrementally-recompiled code is unlikely to ever be as fast as highly-optimized, batch-compiled machine code. In other words, you probably won't ever want to use incremental compilation for your production releases, but it can drastically speed up the development experience, while producing relatively fast code.

Niko spoke about this architecture in his ["Responsive compilers" talk at PLISS 2019][resp]. In that talk he also provided some examples of how the Rust language was accidentally mis-designed for responsive compilation. It is an entirely watchable talk about compiler engineering and I recommend checking it out. 

[resp]: https://www.youtube.com/watch?v=N6b44kMS6OM
[RLS]: https://github.com/rust-lang/rls
[Roselyn]: https://en.wikipedia.org/wiki/.NET_Compiler_Platform
[lsp]: https://langserver.org/



## Tradeoff #6: Build scripts and procedural macros

Cargo allows the build to be customized with two types of custom Rust programs: build scripts and procedural macros. The mechanism for each is different but they both similarly introduce arbitrary computation into the compilation process.

They negatively impact compilation time in several different ways:

First, these types of programs have their own crate ecosystem that also needs to be compiled, so using procedural macros will usually require also compiling crates like [`syn`].

Second, these tools are often used for code generation, and their invocation expands to sometimes large amounts of code.

Third, procedural macros impede distributed build caching tools like [`sccache`]. The reason why surprised me though &mdash; rustc today loads procedural macros as dynamic libraries, one of the few common uses of dynamic libraries in the Rust ecosystem. `sccache` isn't able to cache dynamic library artifacts because it doesn't have visibility into how the linker was invoked to create the dynamic library. So using sccache to build a project that heavily relies on procedural macros will often not speed up the build.

[`syn`]: https://crates.io/crates/syn
[`sccache`]: https://github.com/mozilla/sccache


## Tradeoff #7: Static linking

This one is easy to overlook but has potentially significant impact on the
hack-test cycle. One of the things that people love most about Rust &mdash; that
it produces a single static binary that is trivial to deploy, also requires the
Rust compiler to do a great deal of work linking that binary together.

Every time you build an executable `rustc` needs to run the linker. That
includes every time you rebuild to run a test. In the same experiment I did to
calculate the amount of build time spent in LLVM, Rust spent 11% of debug build
time in the linker. Surprisingly, in release mode it spent less than 1% of time
linking, excluding LTO.

With dynamic linking the cost of linking is deferred until runtime, and parts
of the linking process can be done lazily, or not at all, as functions are
actually called.


## All that stuff summarized

There are a great many factors involved in determining the compile-time of a
compiler, and the resulting run-time performance of its output. It is a miracle
that optimizing compilers ever terminate at all, and that their resulting code
is so amazingly fast. For humans, predicting how to organize their code to find
the right balance of compile time and run time is pretty much impossible.

The size and organization of compilation units has a huge impact on compile
times, and in Rust it is difficult to control compilation unit size, and
difficult to create compilation units that can build in parallel. Internal
compiler parallelism currently does not make up for loss of parallelism between
compilation units.

A variety of factors cause Rust to have a poor build-test cycle, including the
generics model, linking requirements, and the compiler architecture.


## In the next episode of Rust Compile-time Adventures with TiKV

In the next episode of this series we'll do an experiment to illustrate the tradeoffs between dynamic and static dispatch in Rust.

Stay Rusty, friends.


## Thanks

A number of people helped with this blog series. Thanks especially to Niko Matsakis, Graydon Hoare, and Ted Mielczarek for their insights, and Calvin Weng for proofreading and editing.




<!--
### TODO

- https://raphlinus.github.io/rust/2019/08/21/rust-bloat.html
- https://blog.mozilla.org/nnethercote/2019/10/11/how-to-speed-up-the-rust-compiler-some-more-in-2019/
- static linking?
- https://groups.google.com/forum/#!topic/mozilla.dev.platform/-5Ktzq74KTc
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

<!--

TODO:
- monomorphizations are shared
  - https://github.com/rust-lang/rust/issues/47317

-->

<!--
- Comment Link: https://news.ycombinator.com/item?id=22197082
- Comment Link: https://www.reddit.com/r/rust/comments/ew5wnz/the_rust_compilation_model_calamity/
- Comment Link: https://lobste.rs/s/xup5lo/rust_compilation_model_calamity
-->

<!--

ea0fd4d7688d1d462a94cfe4f7ab4d7cb0b30ab5

cargo build --release

real    20m19.025s
user    64m22.746s
sys     2m14.722s

RUSTC_WRAPPER=sccache cargo build --release

1968.13user 104.90system 25:00.42elapsed

RUSTC_WRAPPER=sccache cargo build --release

1974.02user 104.05system 19:20.63elapsed

- https://wiki.alopex.li/WhereRustcSpendsItsTime

-->>

<!--

I cannot make a simple argument about this because I'm still not smart enough
about module systems — the full thing is laid out in dreyer's thesis
(https://www.cs.cmu.edu/~rwh/theses/dreyer.pdf) and discussed in shorter
slide-deck form here (http://macqueenfest.cs.uchicago.edu/slides/dreyer.pdf) —
but suffice to say that recursive modules make it possible to see the "same"
opaque type through two paths that should probably be considered equal but
aren't easily determined to be so, I think in part due to the mix of opacity
that modules provide and the fact that you have to partly look through that
opacity to resolve recursion. so anyway I decided this was probably getting into
"research" and I should just avoid the problem space, go with acyclic modules.


-->
