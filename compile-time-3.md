### Testing the impact of monomorphization on Rust compile times

As I said before, I don't think there's much solid research into the actual impact of dynamic vs. static dispatch on compile times &mdash; just a lot of inherited wisdom (but leave comments if you have good links and I'll update this).

So let's do an experiment!

Obviously, this isn't going to be the definitive word on the subject, just a quick, Rust-specific experiment, but now I'm curious, and I want to give you some numbers to back this all up.

I want to measure the effect of increasing numbers of generic function instantiations on both compile-time and run-time. Due to the contrast between static call inlining vs. indirect call runtime overhead, I suspect there are multiple interesting cases, so I want to try both "deep" and "wide" instantiations, with deep codebases having a deep call graph with few generic type instantiations, and wide codebases a shallow call graph with many instantiations; add to that "in-between" cases with middling call graph depth and type instantiations.

So my experiment is going to be built around a code generator with two inputs: call-graph depth, generic type width. It will generate code like the following.

For static dispatch:

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
impl Io for T1 { fn do_io(&self) { black_box(self) } }

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
