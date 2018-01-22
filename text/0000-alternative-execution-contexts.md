- Feature Name: alternative_execution_contexts
- Start Date: 2018-01-16
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

This is an *experimental RFC* for adding the ability to integrate custom
test/bench/etc frameworks in Rust.

# Motivation
[motivation]: #motivation

Currently, Rust lets you write unit tests with a `#[test]` attribute. We
also have an unstable `#[bench]` attribute which lets one write
benchmarks.

In general it's not easy to use your own testing strategy. Implementing
something that can work within a `#[test]` attribute is fine
(`quickcheck` does this with a macro), but changing the overall strategy
is hard. For example, `quickcheck` would work even better if it could be
done as:

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

A profiling framework may want to use this mode to instrument the binary
in a certain way. We can already do this via proc macros, but having it
hook through `cargo test` would be neat.

Overall, it would be good to have a generic framework for post-build
steps that can support use cases like `#[test]` (both the built-in one
and quickcheck), `#[bench]` (both built in and custom ones like
[criterion]), `examples`, and things like fuzzing. While we may not
necessarily rewrite the built in test/bench/example infra in terms of
the new framework, it should be possible to do so.

The main two problems that we need to solve are:

 - Having a nice API for generating test binaries
 - Having good `cargo` integration so that custom tests are at the same
   level of integration as regular tests as far as build processes are
   concerned

 [cargo-fuzz]: https://github.com/rust-fuzz/cargo-fuzz
 [criterion]: https://github.com/japaric/criterion.rs
 [Compiletest]: http://github.com/laumann/compiletest-rs

# Detailed proposal
[detailed-proposal]: #detailed-proposal

(As an eRFC I'm excluding the "how to teach this" for now; when we have
more concrete ideas we can figure out how to frame it.)

## Test framework proc macro

A test framework is essentially a whole-crate proc macro. Test
frameworks would basically look like this:

```rust
extern crate proc_macro;

use proc_macro::TokenStream;

#[custom_test_framework(attrs(my_test, should_panic))]
pub fn my_bench(input: TokenStream) -> TokenStream {
    // collect all `#[my_test]` functions and generate a `main()` that
    // calls it, replacing the existing `main()` if it exists
}
```

I'm not certain if the test framework itself needs to have a name; it
just needs to register attributes it cares about. However the above
could also have been written as `#[custom_test_framework(my_test,
attrs(should_panic))]`

Because the proc macro is only loaded whilst testing if you use custom
attributes your tests should probably be kept behind `#[cfg(test)]` so
that you don't get unknown attribute warnings whilst loading. (We could
change this by asking attributes to be registered in Cargo.toml, but I
don't find this necessary)

## Helper crate

The proc macro is pretty bare-bones as is. It doesn't help with any
common tasks wrt testing, for example users would still have to do work
like collecting all the test functions or removing `main()`.

For this purpose we provide a helper crate, maintained in the nursery
(or externally), that essentially provides the functionality of
`TestHarnessGenerator` and `EntryPointCleaner` in libsyntax, but instead
works with `syn` or an equivalent AST manipulation library.

We provide the following APIs:

```rust
fn clean_entry_point(tree: syn::ItemMod) -> syn::ItemMod;

trait TestCollector {
    fn fold_function(&mut self, path: syn::Path, func: syn::ItemFn) -> syn::ItemFn;
}

fn collect_tests<T: TestCollector>(collector: &mut T, tree: syn::ItemMod) -> ItemMod;
```

The `TestCollector` lets crates both collect all the test functions that
need calling, as well as also transforming them in any way they wish.
Strictly speaking transforming the functions can be done in a different
pass, but it's convenient to do them in the same pass.

It may be worth generalizing this so that you can also collect entire
modules or something, and providing a simple API for reinserting a main
function into the code.

The in-built `TestHarnessGenerator` does a bunch of reexporting so that
privacy works. As a first pass, I think it's sufficient for
`TestCollector` to mark all modules public, but we can build something
closer to `TestHarnessGenerator` if we wish.

This crate will be out of tree and versioned, so settling on API design
for this _now_ isn't that pressing an issue.


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
