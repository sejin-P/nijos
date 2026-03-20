## Available Tools

### exec
Run shell commands. Allowed: cargo, rustup, qemu-system-x86_64, 
nasm, objdump, readelf, cat, ls, find, grep.
NEVER use: rm -rf, sudo, dd, curl, wget.

### file
Read and write project files within ~/rust-os-project only.

### web_search
Search for Rust OS development documentation, crate docs, 
OSDev wiki references.

### exec
Run shell commands. Allowed: cargo, rustup, qemu-system-x86_64,
nasm, objdump, readelf, cat, ls, find, grep, git.
NEVER use: rm -rf, sudo, dd, curl (except git push/pull), wget.

```

## Project structure
```
~/nijos/
├── AGENTS.md          # agent definitions
├── TOOLS.md           # tool definitions
├── CLAUDE.md          # project context
├── ARCHITECTURE.md    # file created by archtect
├── Cargo.toml
├── .cargo/config.toml
├── src/
│   ├── main.rs
│   ├── kernel/
│   ├── drivers/
│   └── hal/
├── tests/
└── docs/
