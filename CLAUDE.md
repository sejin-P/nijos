# Rust OS Project

## Overview
A bare-metal OS written in Rust targeting x86_64.

## Directory Ownership
| Directory | Owner Agent | Permission |
|-----------|------------|------------|
| docs/, ARCHITECTURE.md | architect | read/write |
| src/kernel/ | kernel-dev | read/write |
| src/drivers/, src/hal/ | driver-dev | read/write |
| tests/ | reviewer | read/write |
| src/main.rs, Cargo.toml | architect | write (others read-only) |

## Build & Run
cargo build
cargo bootimage
qemu-system-x86_64 -drive format=raw,file=target/x86_64-unknown-none/debug/bootimage-rust-os.bin

## Conventions
- Every `unsafe` block MUST have a `// SAFETY:` comment
- All public APIs need doc comments
- No panicking in kernel code — use Result<T, E>
- Run `cargo clippy -- -D warnings` before considering work done
