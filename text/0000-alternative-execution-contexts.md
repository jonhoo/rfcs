- Feature Name: alternative_execution_contexts
- Start Date: 2018-01-16
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

This is an *experimental RFC* for adding support for alternative
execution contexts to Rust. The primary goal of these is to enable easy,
flexible, and seamless integration of custom test/bench/etc. frameworks.

# Motivation
[motivation]: #motivation

Currently, Rust has a small number of supported execution contexts. The
default one (i.e., the one you get with `cargo run`) is to run the
`main()` function defined in a file designated as a binary by the user
under the `[[bin]]` section of their `Cargo.toml`. `test` and `bench`
are two other execution contexts that generate their own `main()`
functions that calls all functions annotated with `#[test]` and
`#[bench]` in the crate's source respectively.

However, it would be good if users could also write crates that define
*new* execution contexts (e.g., `cargo fuzz`), or edit existing ones
(e.g. `cargo test` running quickcheck). This is difficult to do today,
especially in a way that integrates nicely with existing tooling (cargo
in particular).

Implementing something that can work within a `#[test]` attribute is
fine (`quickcheck` does this with a macro), but changing the overall
strategy is hard. For example, `quickcheck` would work even better if it
could be done as:

```rust
#[quickcheck]
fn test(input1: u8, input2: &str) {
    // ...
}
```

If you're trying to do something other than testing, you're out of luck
-- only tests, benches, and examples get the integration from `cargo`
for building auxiliary binaries the correct way. [cargo-fuzz] has to
work around this by creating a special fuzzing crate that's hooked up
the right way, and operating inside of that. Ideally, one would be able
to just write fuzz targets under `fuzz/`.

[Compiletest] (rustc's test framework) would be another kind of thing
that would be nice to implement this way. Currently it compiles the test
cases by manually running `rustc`, but it has the same problem as
cargo-fuzz where getting these flags right is hard. This too could be
implemented as a custom test framework.

A profiling framework (like [criterion]) may want to use this mode to
instrument the binary in a certain way. We can already do this via proc
macros, but having it hook through `cargo test` would be neat.

 [cargo-fuzz]: https://github.com/rust-fuzz/cargo-fuzz
 [criterion]: https://github.com/japaric/criterion.rs
 [Compiletest]: http://github.com/laumann/compiletest-rs

# Detailed proposal
[detailed-proposal]: #detailed-proposal

(As an eRFC I'm excluding the "how to teach this" for now; when we have
more concrete ideas we can figure out how to frame it.)

This eRFC proposes adding a notion of *alternative execution contexts*
that can support use cases like `#[test]` (both the built-in one and
quickcheck), `#[bench]` (both built in and custom ones like
[criterion]), `examples`, and things like fuzzing. While we may not
necessarily rewrite the built in test/bench/example infra in terms of
the new framework, it should be possible to do so.

The main two features proposed are:

 - An API for defining a new execution context, including introspection
   into the crate that is using the execution context.
 - A mechanism for `cargo` integration so that custom execution contexts
   are at the same level of integration as `test` or `bench` as far as
   build processes are concerned.

## Procedural macro for a new execution context

A custom execution context is essentially a whole-crate procedural
macro that is evaluated after all other macros in the target crate have
been evaluated. It is passed the `TokenStream` for every element in the
target crate that has a set of attributes the execution context has
registered interest in. Essentially:

```rust
extern crate proc_macro;
use proc_macro::TokenStream;

#[execution_context(with_attrs(test))]
pub fn like_todays_test(elements: &mut [TokenStream]) -> TokenStream {
    // ...
}
```

`elements` here contains the `TokenStream` for every element in the
target crate that has one of the attributes declared in `with_attrs`.
These may be modules, functions, structs, statics, or whatever else the
execution context wants to support. It is allowed to mutate each
`TokenStream` however it wishes. For example, `libtest` would surround
each annotated function with another function that records whether the
test panicked, and returns the test's result. The returned `TokenStream`
will become the `main()` when this execution context is used.

Because this procedural macro is only loaded when it is used as the
execution context, the `#[test]` annotation should probably be kept
behind `#[cfg(test)]` so that you don't get unknown attribute warnings
whilst loading. (We could change this by asking attributes to be
registered in Cargo.toml, but I don't find this necessary)

### Open question

One of the major questions here is how the generated `main()` references
the given elements. For example, consider the following code under the
above execution context:

```
#[test]
fn foo() {}
```

The generated `main()` will at some point need to call `foo`. It can do
this if it knows the name of the function, but this breaks down with
more complex code:

```
#[cfg(test)]
mod tests {
    #[test]
    fn foo() {}
}
```

Here, the generated `main` will need to call `tests::foo()`. How does it
learn the path to the `foo` function? Perhaps this needs to be passed
explicitly along with the `TokenStream`s? Furthermore, there is the
question of visibility: given that `foo` is private to the `tests`
module, how is the generated `main()` (which is in the same crate, but
not the same module) allowed to reference it?

## Cargo integration

This is a pretty important part of this RFC. Essentially, we need these
to be built correctly, with the correct dependencies all linked in.

Firstly, it's worth noting that there are two kinds of tests. We can
place tests in-tree under `src/` in any regular source file. These are
usually unit tests. We can also place tests under `tests/`. (or
`benches/` for benchmarking, or `examples/`, etc). You can run a
specific one of these via `cargo test --test filename`. You can run only
the unit tests via `cargo test --lib`

You can also declare custom test targets via `[[test]]` or `[[bench]]`
or `[[example]]`

Not all test runners will want the ability to run on in-crate unit
"tests". For example, this may not make sense for `cargo-fuzz`, which
runs targets indefinitely or for a long time; so running fuzz targets in
sequence is a bit weird. Similarly, you don't want examples to be
in-tree; a hypothetical example-running framework would only apply to
files under `examples/`.

(The ability to exclude the unit test model isn't necessary -- it can be
worked around -- but it would be nice to have)

In general, test framework proc macros declare themselves in
`Cargo.toml`:


```toml
[package]
name = "my-test-framework"
[lib]
test-framework = true
test-framework-allow-unit = true # if false, disallow running as unit tests on src/ . perhaps should be called allow-lib
```


They can also declare `harness-dependencies`. Our built in test
harnesses depend on `libtest`, which always exists in your rust install.
Other test harnesses will probably want to be able to depend on the
runtime component of the test harness, or depend on other helper crates
(e.g. colorful terminal output).

When building a test, the `harness-dependencies` of the current custom
test framework will be merged with `dev-dependencies` of the crate being
used (overlaps will be semver-merged; if this is not possible it will
fail to compile) and the combined set will be used together as
dev-dependencies.

```toml
[harness-dependencies]
my-test-framework-rt = "1.0"
```

Crates _using_ a test framework need to declare them in the Cargo.toml:

```toml
[testing.framework.test]
framework = { my-test-framework = "1.0" }
folder = "tests/"

[testing.framework.fuzz]
framework = { rust-fuzz = "1.0" }
folder = "fuzzers/"
```

These can be invoked via `cargo test --kind test` and `cargo test --kind
fuzz`. `testing.test`.


Custom test targets can be declared via `[[testing.target]]`

```toml
[[testing.target]]
framework = fuzz
path = "foo.rs"
name = "foo"
```

`[[test]]` is an alias for `[[testing.target]] framework = test` (same
goes for `[[bench]]` and `[[example]]`)

By default, the crate has an implicit "test", "bench", and "example"
framework that use the default libtest stuff. (example is a no-op
framework that just runs stuff). However, declaring a framework with the
name `test` will replace the existing `test` framework. In case you wish
to supplement the framework, use a different name.

By default, `cargo test` will run doctests and the `test` and `examples`
framework. This can be customized:

```toml
[[testing.kinds]]
tests = [test, quickcheck, examples]
bench = [bench, criterion]
```

This means that `cargo test` will, aside from doctests, run `cargo test
--kind test`, `cargo test --kind quickcheck`, and `cargo test --kind
examples` (and similar stuff for `cargo bench`).

The generated test binary should be able to take one identifier
argument, used for narrowing down what tests to run. I.e. `cargo test
--kind quickcheck my_test_fn` will build the test(s) and call them with
`./testbinary my_test_fn`. Typically, this argument is used to filter
tests further; test harnesses should try to use it for the same purpose.


## To be designed

This contains things which we should attempt to solve in the course of
this experiment, for which this eRFC does not currently provide a
concrete proposal.

### Standardizing the output

We should probably provide a crate with useful output formatters and
stuff so that if test harnesses desire, they can use the same output
formatting as a regular test. This also provides a centralized location
to standardize things like json output and whatnot.

@killercup is working on a proposal for this which I will try to work in.

### Configuration

Currently we have `cfg(test)` and `cfg(bench)`. Should `cfg(test)` be
applied to all? Should `cfg(nameofharness)` be used instead? Ideally
we'd have a way when declaring a framework to declare what cfgs it
should be built with.


# Rationale and alternatives
[alternatives]: #alternatives

 - We should either do this or stabilize the existing bencher, not doing
   both would mean that things that
 - A procedural macro based solution might be too general, instead
   exposing a more restricted "collect all functions and generate a main
   function" API might be best. However, this will likely use procedural
   macros under the hood anyway, so it's not more effort to implement
   this as a procedural macro with the "collect test functions" stuff
   coming from helper APIs maintained out of tree.

# Unresolved questions
[unresolved]: #unresolved-questions

 - The TBD section should be resolved through implementation and
   experimentation
 - The general syntax and toml stuff should be approximately settled on
   before this eRFC merges, but iterated on later
 - The details of the helper library should be resolved through
   implementation and experimentation
 - Should we be shipping a built in bencher at all? Could we instead
   default `cargo bench` to a `rust-lang-nursery` crate?
 - Does this run before or after regular macro expansion? (Probably
   after)
