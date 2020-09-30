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

You need to install a version of rustc more recent than 1.43.1 with the
`x86_64-unknown-linux-musl` target:

    rustup install 1.46.0
    rustup default 1.46.0
    rustup target add x86_64-unknown-linux-musl

You'll need the following structure and branches:

- [apps/helloworld-rust, csoldani/rust](https://github.com/cffs/app-helloworld-rust)
- [libs/compiler-rt, csoldani/rust](https://github.com/cffs/lib-compiler-rt/tree/csoldani/rust)
- [libs/libcxx, csoldani/rust](https://github.com/cffs/lib-libcxx/tree/csoldani/rust)
- [libs/libcxxabi, staging](https://github.com/unikraft/lib-libcxxabi/)
- [libs/librust, staging](https://github.com/cffs/lib-librust/tree/staging)
- [libs/libunwind, staging](https://github.com/unikraft/lib-libunwind/)
- [libs/musl, ggain/features](https://github.com/cffs/lib-musl/tree/ggain/features)
- [unikraft, fix_tls_align_issue](https://github.com/cffs/unikraft/tree/fix_tls_align_issue)

In `apps/helloworld-rust`, you'll need to do a `make menuconfig`. Select
desired targets (`linuxu` or `kvm`) and `libcxx` (required by `libunwind`).

You should then be able to build the application with `make`.

If the application blocks after printing "Welcome to  ",
disable the `ukboot` banner in `make menuconfig`.

## Limitations

The port of musl on unikraft is ongoing work, and is not fully functional. More
specifically, musl uses some Linux syscalls which have not yet been re-routed
through the syscall shim layer, or for which the functionality has not yet been
implemented.

The KVM platform currently does not work properly, due to relocation linking
problems. The Xen platform has not been tested.
