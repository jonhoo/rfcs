- Feature Name: alternative_execution_contexts
- Start Date: 2018-01-22
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

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

As an eRFC I'm excluding this section for now; when we have more
concrete ideas we can figure out how to frame it.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

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
pub fn like_todays_test(items: &[AnnotatedItem]) -> TokenStream {
    // ...
}
```

where

```rust
struct AnnotatedItem
    tokens: TokenStream,
    span: Span,
    attributes: TokenStream,
    path: SomeTypeThatRepresentsPathToItem
}
```

`items` here contains an `AnnotatedItem` for every element in the
target crate that has one of the attributes declared in `with_attrs`.
An execution context could declare that it reacts to multiple different
attributes, in which case it would get all items with any of the
listed attributes. These items be modules, functions, structs,
statics, or whatever else the execution context wants to support. Note
that the execution context function can only see all the annotated
items, not modify them; modification would have to happen with regular
procedural macros The returned `TokenStream` will become the `main()`
when this execution context is used.

Because this procedural macro is only loaded when it is used as the
execution context, the `#[test]` annotation should probably be kept
behind `#[cfg(test)]` so that you don't get unknown attribute warnings
whilst loading. (We could change this by asking attributes to be
registered in Cargo.toml, but I don't find this necessary)

### A note about paths

One of the major questions here is how the generated `main()` references
the given items. For example, consider the following code under the
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
learn the path to the `foo` function? Furthermore, there is the question
of visibility: given that `foo` is private to the `tests` module, how is
the generated `main()` (which is in the same crate, but not the same
module) allowed to reference it? Perhaps the compiler needs to change
the visibility of ancestors of items annotated with an attribute
registered by the chosen execution context?

## Cargo integration

Alternative execution contexts need to integrate with cargo.
In particular, when crate `a` uses a crate `b` which provides an
execution context, `a` needs to be able to specify when `b`'s execution
context should be used. Furthermore, cargo needs to understand that when
`b`'s execution context is used, `b`'s dependencies must also be linked.
Note that `b` could potentially provide multiple execution contexts ---
these are named according to the name of their `#[execution_context]`
function.

Nothing special is required in `Cargo.toml` for a crate that provides an
execution context. Any exported function annotated with
`#[execution_context]` can be used as an execution context in a
dependent crate. Dependencies of defined execution contexts should be
listed in `[dependencies]` of the crate that defines the context.

For crates that wish to *use* a custom execution context, they do so by
defining a new execution context under a new `execution` section in
their `Cargo.toml`:

```toml
[execution.context.fuzz]
provider = { rust-fuzz = "1.0" }
folders = ["fuzz/"]
```

This defines an execution context named `fuzz`, which uses the
implementation provided by the `rust-fuzz` crate. When run, it will be
applies to all files in the `fuzz` directory. By default, the following
executions are defined:

```toml
[execution.context.test]
provider = { test = "1.0", context = "test" }
folders = ["tests/", "src/"]

[execution.context.bench]
provider = { test = "1.0", context = "bench" }
folders = ["benchmarks/"]
```

These can be overridden by a crate's `Cargo.toml`. The `context`
property is used to disambiguate when a single crate has multiple
functions tagged `#[execution_context]` (if we were using the example
execution provider further up, we'd give `like_todays_test` here).
`test` here is `libtest`, though note that it could be maintained
out-of-tree, and shipped with rustup.

When building a particular execution context, the `dependencies` of the
crate providing the execution context is merged with the
`dev-dependencies` of the target crate (overlaps will be semver-merged;
if this is not possible it will fail to compile) and the combined set
will be used together as dev-dependencies.

To invoke a particular execution context, a user invokes `cargo execute
<context>`. `cargo test` and `cargo bench` are aliases for `cargo
execute test` and `cargo execute bench` respectively. Any additional
arguments are passed to the execution context binary. By convention, the
first position argument should allow filtering which
test/benchmarks/etc. are run.

## Execution context sets

For many crates, it would be attractive to have `cargo test` run
multiple different testing-oriented execution contexts at once. This can
be achieved by defining an execution context *set*:

```toml
[execution.set.test]
contexts = [test, quickcheck, examples]
```

`cargo execute foo` will first see if a set is defined with the name
`foo`, and if so, execute all of its contexts. If not, it will look for
a context named `foo`, and if it exists, execute it as outlined above.

# Drawbacks
[drawbacks]: #drawbacks

 - This adds more sections to `Cargo.toml`.
 - This complicates the execution path for cargo, in that it now needs
   to know about execution contexts and sets.
 - Flags and command-line parameters for test and bench will now vary
   between execution contexts, which may confuse users as they move
   between crates.

# Rationale and alternatives
[alternatives]: #alternatives

 - Stabilize `#[bench]` and extend libtest with setup/teardown and other
   requested features. This would complicate the in-tree libtest,
   introduce a barrier for community contributions, and discourage other
   forms of testing or benchmarking.
 - A procedural macro based solution might be too general. We could
   instead expose a more restricted "collect all functions and generate
   a main function" API might be best. However, this will likely use
   procedural macros under the hood anyway, so it's not more effort to
   implement this as a procedural macro with the "collect test
   functions" stuff coming from helper APIs maintained out of tree.

# Unresolved questions
[unresolved]: #unresolved-questions

Beyond the big one listed under the reference-level explanation, there
are other open questions surrounding this RFC. Some of these we should
attempt to solve in the course of this experiment, and this eRFC does
not currently provide a concrete proposal.

## Integration with doctests

Documentation tests are somewhat special, in that they cannot easily be
expressed as `TokenStream` manipulations. In the first instance, the
right thing to do is probably to have an implicitly defined execution
context called `doctest` which is included in the execution context set
`test` by default.

Another argument for punting on doctests is that they are intended to
demonstrate code that the user of a library would write. They're there
to document *how* something should be used, and it then makes somewhat
less sense to have different "ways" of running them.

## Translating existing cargo test flags

Today, `cargo test` takes a number of flags such as `--lib`, `--test
foo`, and `--doc`. As breaking these at this point would make users sad,
cargo should recognize them and map to the appropriate execution
contexts.

## Standardizing the output

We should probably provide a crate with useful output formatters and
stuff so that if test harnesses desire, they can use the same output
formatting as a regular test. This also provides a centralized location
to standardize things like json output and whatnot.

@killercup is working on a proposal for this which I will try to work in.

## Configuration

Currently we have `cfg(test)` and `cfg(bench)`. Should `cfg(test)` be
applied to all? Should `cfg(execution_context_name)` be used instead?
Ideally we'd have a way when declaring an execution context to declare
what cfgs it should be built with.

## Other questions

 - The big one under execution context procedural macros.
 - The general syntax and toml stuff should be approximately settled on
   before this eRFC merges, but iterated on later
 - Should an execution context be able to declare "defaults" for what
   folders and execution sets it should be added to? This might save
   users from some boilerplate in a large number of situations.
 - Should we be shipping a bencher by default at all (i.e., in libtest)?
   Could we instead default `cargo bench` to a `rust-lang-nursery`
   crate?
