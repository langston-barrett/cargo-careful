# cargo-careful

`cargo careful` is a tool to run your Rust code extra carefully -- opting into a bunch of
nightly-only extra checks that help detect Undefined Behavior, and using a standard library with
debug assertions. For example, it will find the error in the following snippet:

```rust
fn main() {
    let arr = [1, 2, 3, 4];
    let slice = &arr[..2];
    let value = unsafe { slice.get_unchecked(2) };
    println!("The value is {}!", value);
}
```

To use `cargo careful`, first install it:

```
cargo install cargo-careful
```

and then run the following in your project:

```
cargo +nightly careful test
```

Running `cargo careful` requires a recent nightly toolchain. You can also `cargo +nightly careful
run` to execute a binary crate. All `cargo test` and `cargo run` flags are supported.

The first time you run `cargo careful`, it needs to run some setup steps, which requires the
`rustc-src` rustup component -- the tool will offer to install it for you if needed.

## What does it do?

### Assertions

The most important thing `cargo careful` does is that it builds the standard library with debug
assertions. The standard library already contains quite a few sanity checks that are enabled as
debug assertions, but the usual rustup distrubtion compiles them all away to avoid run-time checks.
Furthermore, `cargo careful` sets some flags that tell rustc to insert extra run-time checks.

Here are some of the checks this enables:

- `get_unchecked` in slices performs bounds checks.
- `copy`, `copy_nonoverlapping`, and `write_bytes` check that pointers are aligned and non-null and
  (if applicable) non-overlapping.
- `{NonNull,NonZero*,...}::new_unchecked` check that the value is valid.
- `unreachable_unchecked` checks that it actually is not being reached.
- The collection types perform plenty of internal consistency checks.
- `mem::zeroed` and the deprecated `mem::uninitialized` panic if the type does not allow that kind
  of initialization (with a check that is stricter than the default). (This is `-Zstrict-init-checks`.)
- Extra UB-checking is done during const-evaluation. (This is `-Zextra-const-ub-checks`.)

That said, there is a lot of Undefined Behavior that is *not* detected by `cargo careful`; check out
[Miri](https://github.com/rust-lang/miri) if you want to be more exhaustively covered.
The advantage of `cargo careful` over Miri is that it works on all code, supprts using arbitrary system and C FFI functions, and is much faster.

### RUSTFLAGS

`cargo careful` honors the `CARGO_ENCODED_RUSTFLAGS` and `RUSTFLAGS` environment variables as well
as the `build.rustflags` cargo setting (in that order, the first one being set is used). It
currently does *not* honor the `target.rustflags` settings as that would require re-implementing all
the target `cfg` logic from cargo. The flags are applied to *both* the sysroot build and the program
itself.

### Sanitizing

`cargo careful` can additionally build and run your program and standard library
with a sanitizer. This feature is experimental and disabled by default, as
the [underlying `rustc` feature](https://doc.rust-lang.org/nightly/unstable-book/compiler-flags/sanitizer.html)
doesn't play well with proc macros.

To use a sanitizer, pass the command-line flag `-Zcareful-sanitizer=<your_sanitizer>` to `cargo careful`.
The list of supported sanitizers and targets can be found
[here](https://doc.rust-lang.org/nightly/unstable-book/compiler-flags/sanitizer.html).
If you pass `-Zcareful-sanitizer` without specifying a sanitizer, [`AddressSanitizer`](https://clang.llvm.org/docs/AddressSanitizer.html)
will be used.

By default, when using `AddressSanitizer`, `cargo careful` will disable memory leak checking by
setting `ASAN_OPTIONS=detect_leaks=0` in your program's environment, as memory leaks are not
usually a soundness or correctness issue. If you set the `ASAN_OPTIONS` environment variable
yourself (to any value, including an empty string), that will override this behavior.

### `cfg` flag

`cargo careful` sets the `careful` configuration flag, so you can use Rust's compile-time
conditional mechanisms (`#[cfg(careful)]`, `#[cfg_attr(careful, ...)]`, `cfg!(careful)`) to check
whether code is being run carefully.
