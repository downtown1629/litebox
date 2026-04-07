# LiteBox — Getting Started

## Overview

LiteBox is a security-focused library OS / sandboxing platform (owned by Microsoft) that minimizes
the attack surface by drastically reducing the interface to the host OS. It enables running
unmodified Linux programs on various substrates.

This document covers how to build, run, and test LiteBox on Linux userland — the simplest substrate.

---

## Prerequisites

- Rust stable toolchain (project uses `rust-toolchain.toml` pinned to stable)
- `cargo-nextest` for running tests
- `gcc` for compiling test C programs

Verify your environment:

```bash
rustc --version     # tested with 1.94.0
cargo --version
cargo nextest --version
```

---

## Build

Build the Linux userland runner:

```bash
cargo build -p litebox_runner_linux_userland
```

The binary is produced at `./target/debug/litebox_runner_linux_userland`.

Other useful build commands:

```bash
cargo fmt                                   # format
cargo clippy --all-targets --all-features   # lint
cargo nextest run                           # run all tests
```

---

## Running Programs

LiteBox runs programs inside an isolated sandbox with its own in-memory filesystem. The host
filesystem is **not** visible to the guest program — you must explicitly supply any files the
program needs.

### Static Binaries (simplest)

Compile a statically linked binary, then run it with `--rewrite-syscalls` to automatically
rewrite syscall sites before execution:

```bash
# Compile static binary
gcc -static -o /tmp/hello_static hello.c

# Run inside LiteBox
cargo run -p litebox_runner_linux_userland -- \
  -Z --rewrite-syscalls /tmp/hello_static
```

`-Z` enables unstable options (required for `--rewrite-syscalls`).

### Dynamic Binaries

Dynamically linked binaries need their shared libraries available inside the sandbox. The
workflow is:

1. Rewrite the binary and all its shared libraries with `litebox_syscall_rewriter`
2. Bundle the rewritten libraries into a `.tar` rootfs
3. Run with `--initial-files <rootfs.tar>`

```bash
RUNNER=./target/debug/litebox_runner_linux_userland
REWRITER=./target/debug/litebox_syscall_rewriter
TARDIR=/tmp/litebox_rootfs
TARFILE=/tmp/litebox_rootfs.tar

# 1. Rewrite the main binary
$REWRITER /tmp/hello -o /tmp/hello.hooked

# 2. Find, rewrite, and package all shared library dependencies
rm -rf $TARDIR && mkdir -p $TARDIR
for dep in $(ldd /tmp/hello | grep -oP '/[^ ]+' | grep -v 'linux-vdso'); do
    destdir="$TARDIR/$(dirname $dep)"
    mkdir -p "$destdir"
    $REWRITER "$dep" -o "$destdir/$(basename $dep)"
done

# 3. Create the rootfs tar
tar -C $TARDIR -cf $TARFILE .

# 4. Run
$RUNNER -Z \
  --interception-backend rewriter \
  --initial-files $TARFILE \
  --env LD_LIBRARY_PATH=/lib64:/lib32:/lib \
  /tmp/hello.hooked
```

Expected output includes audit log lines followed by program output:

```
[audit] main binary is patched by libOS
[audit] interp=/lib64/ld-linux-x86-64.so.2
...
hello from litebox!
```

---

## CLI Reference

| Flag | Description |
|---|---|
| `-Z` / `--unstable` | Enable unstable options (required for most flags below) |
| `--rewrite-syscalls` | Rewrite syscall sites in the binary before running |
| `--interception-backend rewriter\|seccomp` | How syscalls are intercepted (default: `rewriter`) |
| `--initial-files <path.tar>` | Mount a tar archive as the initial rootfs |
| `--program-from-tar` | Load program binary from the tar instead of the host FS |
| `--env K=V` | Pass an environment variable into the sandbox (repeatable) |
| `--forward-env` | Forward the host's environment variables into the sandbox |
| `--tun-device-name <name>` | Attach to a TUN device for networking |

---

## How the Syscall Rewriter Works

LiteBox intercepts syscalls made by the guest program. There are two backends:

- **rewriter** (default): Rewrites `syscall` instructions in the ELF binary ahead-of-time to
  redirect into LiteBox. Also injects `litebox_rtld_audit.so` to hook dynamically loaded
  libraries at load time via the rtld audit interface.
- **seccomp**: Uses Linux seccomp to intercept syscalls at runtime without binary rewriting.

For the rewriter backend, every ELF (main binary + all shared libs) must be rewritten with
`litebox_syscall_rewriter` before use.

---

## Running the Test Suite

```bash
cargo nextest run -p litebox_runner_linux_userland
```

### Results on this machine

```
7/10 passed (1 slow — python test ~235s)
2 failed — expected:
  • test_node_with_rewriter       → Node.js not installed
  • test_tun_and_runner_with_iperf3 → needs TUN device (CI configures this)
```

The CI pipeline (`.github/workflows/ci.yml`) sets up a TUN device and installs `iperf3` and
`diod` before running tests. On a plain developer machine these two tests are expected to fail.

### Running a specific test

```bash
cargo nextest run -p litebox_runner_linux_userland test_static_exec_with_rewriter
```

---

## Architecture at a Glance

```
┌─────────────────────────────────────────────────────┐
│  Guest program (unmodified Linux ELF)               │
├─────────────────────────────────────────────────────┤
│  litebox_shim_linux  — Linux syscall surface (ABI)  │
├─────────────────────────────────────────────────────┤
│  litebox             — core: fd/fs/mm/net            │
├─────────────────────────────────────────────────────┤
│  litebox_platform_linux_userland  — Platform trait  │
├─────────────────────────────────────────────────────┤
│  litebox_runner_linux_userland    — main() / CLI    │
└─────────────────────────────────────────────────────┘
```

Key source files:

| File | Role |
|---|---|
| [litebox/src/lib.rs](litebox/src/lib.rs) | Public library API |
| [litebox/src/litebox.rs](litebox/src/litebox.rs) | Core `LiteBox` struct (fd tables, global state) |
| [litebox/src/platform/mod.rs](litebox/src/platform/mod.rs) | Central `Platform` trait |
| [litebox_runner_linux_userland/src/lib.rs](litebox_runner_linux_userland/src/lib.rs) | Runner logic, CLI arg handling, FS setup |
| [litebox_runner_linux_userland/src/main.rs](litebox_runner_linux_userland/src/main.rs) | Entry point |
| [litebox_runner_linux_userland/tests/run.rs](litebox_runner_linux_userland/tests/run.rs) | Integration test suite |
