# How to use a rust program in unikraft?

Note that rust support in unikraft is ongoing work.

## Goal

Any rust program/library that you can compile with cargo as a static library
should be supported, provided that:

- it does tries to load libraries dynamically with `dlopen()`;
- it does not issue unsupported syscalls.

I.e., most rust-only code should work, most rust wrappers around other
libraries won't.

## Principle

To build the rust program, we turn it into a library, that exports its main
entry point as a C-callable function (see `helloworld/src/lib.rs`).

We complement it with a simple C `main()` function, that calls that entry
point.

The rust library is built using the `x86_64-unknown-linux-musl` target, asking
for a `staticlib`, i.e. an ar archive. By default, rustc would provide a full
runtime, including the musl C library, in that static library. However, you can
pass it the flag `-C target-feature=-crt-static` to avoid that. In the latter
case, rustc will include the rust runtime, but not the C runtime it depends on.
This allows us to latter link the unikraft runtime (with its musl port) instead
of the rust-provided musl one. This flag can be passed through `RUSTFLAGS`
environment variable, or through `.cargo/config` in your rust project (see
`helloworld/.cargo/config`).

Once the rust components have been built as static libraries, they can be added
to your unikraft project through `UKALIBS-y`.

## How to use this example

You need to install rustc 1.43.1 with the `x86_64-unknown-linux-musl` target:

    rustup install 1.43.1
    rustup default 1.43.1
    rustup target add x86_64-unknown-linux-musl

You'll need the following structure and branches:

    apps/helloworld-rust    csoldani/rust
    libs/compiler-rt        csoldani/rust
    libs/libcxx             csoldani/rust
    libs/libcxxabi          staging
    libs/librust            staging
    libs/libunwind          staging
    libs/musl               ggain/features
    unikraft                staging

In `apps/helloworld-rust`, you'll need to do a `make menuconfig`. Select
desired targets (`linuxu` or `kvm`) and `libcxx` (required by `libunwind`).

You should then be able to build the application with `make`.

If the application blocks after printing "Welcome to  ",
disable the `ukboot` banner in `make menuconfig`.

## Limitations

The port of musl on unikraft is ongoing work, and is not fully functional. More
specifically, musl uses some Linux syscalls which have not yet been re-routed
through the syscall shim layer, or for which the functionality has not yet been
implemented. Moreover, the thread-local storage (TLS) support is not
functional. Currently, the TLS initialisation code of musl is not called at
all, which can break musl programs using TLS as musl makes some assumptions
which are not true in unikraft. This is especially problematic with rust
programs, which are extremely cautious with thread-safety, and rely a lot on
TLS.

Moreover, this rust support is fragile, as it depends on very specific code
generation parameters. In particular, it prevents the use of rustc versions
after 1.43.1, as version 1.44.0 dropped one such parameter: `no-integrated-as`
(asking rustc to use GNU as instead of LLVM). Without those code generation
parameters, we run into relocation issues that can make final linking to fail,
or generate crashes at runtime. All observed issues are related to TLS, so that
we hope they will disappear once TLS support in external static libraries is
fully functional.
