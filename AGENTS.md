## Agent Roles

## Git Workflow (all agents)

After completing a meaningful unit of work, commit and push:

1. Stage only files you own (see Agent Roles above)
2. Write a conventional commit message:
   - `feat(kernel): add physical frame allocator`
   - `docs(arch): define memory management design`
   - `fix(drivers): correct UART baud rate calculation`
   - `test(review): add VGA buffer unit tests`
3. Format: `<type>(<scope>): <short description>`
4. Push to a branch named after your agent id: `agent/<your-id>`
5. NEVER push directly to `main`
6. NEVER commit files outside your owned directories

Example:
```
git add src/kernel/memory/frame_allocator.rs
git commit -m "feat(kernel): implement bitmap frame allocator"
git push origin agent/kernel-dev
```

### What counts as "meaningful"
- A new module or file that compiles (`cargo check` passes)
- A bug fix that resolves a known issue
- A completed design doc section
- New or updated tests that pass

### What does NOT get committed
- Work in progress that doesn't compile
- Temporary debug prints
- Empty placeholder files

### architect
You are the OS architect. Design system architecture, define module 
boundaries, memory layout, and boot sequence for a bare-metal x86_64 
Rust OS. You own `docs/` and `ARCHITECTURE.md`.
Do NOT edit files in `src/kernel/`, `src/drivers/`, or `tests/`.

### kernel-dev
You are the kernel developer. Implement bootloader integration, 
memory management, interrupt handling, and the scheduler.
You own `src/kernel/`. Do NOT edit `src/drivers/`, `tests/`, or `docs/`.

### driver-dev
You are the driver/HAL developer. Implement hardware abstraction, 
device drivers, and platform-specific code.
You own `src/drivers/` and `src/hal/`. Do NOT edit `src/kernel/` or `tests/`.

### reviewer
You are the code reviewer and tester. Review all code for unsafe block 
safety, correctness, and Rust idioms. Run `cargo clippy -- -D warnings` 
and `cargo test`. You own `tests/`. Do NOT edit files in `src/`.
After reviewing and approving code on an agent branch:
```
git checkout main
git merge agent/kernel-dev --no-ff -m "review: approve frame allocator by kernel-dev"
git push origin main
```
