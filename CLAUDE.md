# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

LiteBox is a Microsoft-owned, security-focused library OS (sandboxing platform) written in Rust. It reduces attack surface by minimizing the host OS interface, enabling unmodified Linux programs to run on various substrates (Windows, Linux, SEV SNP, OP-TEE, LVBS). Pre-1.0, actively evolving.

## Architecture

LiteBox uses a **North-South pattern**:

- **North (guest-facing)**: Shims that provide a familiar ABI (Linux syscalls, OP-TEE) to guest programs
  - `litebox_shim_linux`, `litebox_shim_optee`
- **Core**: `litebox` crate — `#[no_std]` library with fd/fs/mm/net abstractions and the `Platform` trait
- **South (host-facing)**: Platform implementations that fulfill the `Platform` trait on different hosts
  - `litebox_platform_linux_userland`, `litebox_platform_windows_userland`, `litebox_platform_linux_kernel`, `litebox_platform_lvbs`, `litebox_platform_multiplex`
- **Runners**: Entry-point binaries that wire a North shim to a South platform
  - `litebox_runner_linux_userland` (most common for development), `litebox_runner_linux_on_windows_userland`, `litebox_runner_lvbs`, `litebox_runner_snp`, `litebox_runner_optee_on_linux_userland`
- **Tools**: `litebox_syscall_rewriter` (AOT rewrites ELF `syscall` instructions to trampolines), `litebox_packager` (self-contained ELF packaging), `litebox_rtld_audit` (hooks dynamically loaded libraries)

### Syscall Interception

Two backends: **Rewriter** (default, AOT rewriting — faster) and **Seccomp** (runtime interception — works on stock binaries).

## Build & Development

Stable Rust toolchain (pinned in `rust-toolchain.toml`). LVBS and SNP runners require nightly with custom targets.

```bash
# Format (required before every commit)
cargo fmt

# Build
cargo build
cargo build -p litebox_runner_linux_userland   # single crate

# Lint — pedantic clippy enabled at workspace level
cargo clippy --all-targets --all-features

# Test (primary runner is nextest)
cargo nextest run                              # all tests
cargo nextest run -p litebox_runner_linux_userland  # single crate
cargo nextest run -p litebox_runner_linux_userland test_static_exec_with_rewriter  # single test
cargo test --doc                               # doc tests (nextest doesn't support these)

# 32-bit (CI validates both x86-64 and i686)
cargo nextest run --target=i686-unknown-linux-gnu

# LVBS build (nightly, custom target)
cargo +nightly build -Z build-std-features=compiler-builtins-mem -Z build-std=core,alloc \
  --manifest-path=litebox_runner_lvbs/Cargo.toml --target litebox_runner_lvbs/x86_64_vtl1.json

# Documentation
cargo doc --no-deps --all-features --document-private-items
```

## CI Requirements

CI is defined in `.github/workflows/ci.yml`. All of these must pass:

- `cargo fmt --check` — formatting compliance
- `cargo clippy --all-targets --all-features` with `-Dwarnings` — no warnings
- `cargo nextest run` on x86-64 and i686
- `cargo test --doc` — doc tests
- `cargo doc` with `-Dwarnings` — documentation builds cleanly
- `no_std` verification: core crates build with `--target x86_64-unknown-none`
- Separate jobs for LVBS (nightly), SNP (nightly), and Windows builds

## Code Guidelines

From `.github/copilot-instructions.md`:

- **`unsafe`**: Minimize usage. Every `unsafe` block must have a safety comment explaining soundness.
- **`no_std`**: Core `litebox` crate must remain `no_std`. Favor `no_std` in new crates.
- **Dependencies**: Justify additions; use `default-features = false`.
- **Testing**: Write unit tests for new functionality. Integration tests live in each runner's `tests/` directory.
- **Documentation**: Document all public APIs and non-trivial logic.

## Workspace Notes

- `litebox_runner_lvbs` and `litebox_runner_snp` are excluded from default workspace members (require nightly + custom targets)
- `litebox_shim_optee` is in default-members but not in workspace `members` list directly (included via `litebox_runner_optee_on_linux_userland`)
- `dev_tests` and `dev_bench` are CI-only crates not meant for release
- Nextest config in `.config/nextest.toml`: TUN tests serialize (max-threads=1), CI profile has retries for known flaky tests
- `bacon.toml` provides convenient watch-mode jobs (`c` for clippy, `n` for nextest, `d` for docs)
