# Actually Portable Executables with Cosmopolitan Libc and Rust

This repository contains a simple `Hello world!` example in the [Rust][rust]
programming language, that builds with [Cosmopolitan Libc][cosmo]. 

I created a [custom compilation target][custom-target] for Rust, called
`x86_64-unknown-linux-cosmo`, to provide a build process that uses the
Cosmopolitan Libc amalgamation and `cargo`. I followed the documentation in the
[Rust Embedonomicon][custom-embed] to create the target.

An alternative method to build APEs with Rust would be to avoid `cargo`, just
use `rustc` or equivalent compiler to generate `.o` files, and then write a
shell script that does the linking with the expected flags. I have not tried
this method.

> a previous attempt at a Rust APE was done without using the `std` crate. You
can view that in the [`without-std` branch of this repo][without-std-branch].

## Building a Rust APE with the `std` crate

1. Download the Cosmopolitan Libc [amalgamation][amalg-download] into the `libcosmo` folder:

```bash
cd libcosmo
wget https://justine.lol/cosmopolitan/cosmopolitan.zip
unzip cosmopolitan.zip
cd ../
```

For reference, I used the nightly version of `cosmopolitan.a` from June 22 2022,
which can be built from source if needed from [this commit][cosmo-nightly].

2. Download the necessary host toolchain and source code for Rust:

```bash
# I was on Debian 11, so I did this
rustup toolchain install nightly-x86_64-unknown-linux-gnu
rustup component add rust-src --toolchain nightly-x86_64-unknown-linux-gnu
# on Alpine Linux, you may need to do
rustup toolchain install nightly-x86_64-unknown-linux-musl
rustup component add rust-src --toolchain nightly-x86_64-unknown-linux-musl
```

For reference, I used the nightly version of Rust from June 22 2022.

3. run `cargo build` to get the debug executable. This uses a bash script that
   removes unnecessary linker arguments. A recent version of `gcc` and `ld.bfd`
   is required.

```bash
cargo +nightly build -Zbuild-std=panic_abort,std -Zbuild-std-features=""  --target=./x86_64-unknown-linux-cosmo.json
```

For reference, I used the below versions of `gcc` and `ld.bfd`

```
gcc (Debian 10.2.1-6) 10.2.1 20210110
Copyright (C) 2020 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

```
GNU ld (GNU Binutils for Debian) 2.35.2
Copyright (C) 2020 Free Software Foundation, Inc.
This program is free software; you may redistribute it under the terms of
the GNU General Public License version 3 or (at your option) a later version.
This program has absolutely no warranty.
```

4. run `objcopy` on the debug binary to obtain the APE:

```bash
# objcopy is the same version as ld.bfd above
objcopy -SO binary ./target/x86_64-unknown-linux-cosmo/debug/hello_world.com.dbg ./hello_world.com
# run the APE
./hello_world.com
# see syscalls made by the APE
./hello_world.com --strace
```

Now we have an Actually Portable Executable built with Rust! There might some
edge cases that I haven't noticed, so clone/fork the repo and try it out!  The
[`without-std` branch][without-std-branch] of this repo might be useful.

## TODOs

- [ ] figure out build config to avoid using `libunwind`

The `std` crate relies on
[`backtrace`](https://github.com/rust-lang/backtrace-rs), which depends on
[`libunwind`](https://github.com/libunwind/libunwind) in the default builds for
unix. To work around this, `cosmopolitan.a` currently has stubs for the
functions that `backtrace` relies on. However, it might be easier to provide a
build flag in `Cargo.toml` to use the `noop` module of `backtrace`. 

A small change needs to be submitted to the source code of `backtrace` (in the
`cfg_if!`
[here](https://github.com/rust-lang/backtrace-rs/blob/4e5a3f72929f152752d5659e95bb15c8f6b41eff/src/backtrace/mod.rs#L128))
to allow choosing `noop` when building as part of the `std` crate. This
conditional compilation flag should be accessible when building the `std` crate
either via `Cargo.toml` or something like `-Z use-std-backtrace-noop` in the
build command.

[without-std-branch]: https://github.com/ahgamut/rust-ape-example/tree/without-std
[rust]: https://rust-lang.org
[cosmo]: https://github.com/jart/cosmopolitan
[cosmo-nightly]: https://github.com/jart/cosmopolitan
[amalg-download]: https://justine.lol/cosmopolitan/download.html
[custom-target]: https://doc.rust-lang.org/rustc/targets/custom.html
[custom-embed]: https://docs.rust-embedded.org/embedonomicon/custom-target.html
