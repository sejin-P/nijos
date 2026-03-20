# ARCHITECTURE.md - NijOS Design Specification

**Target:** x86_64 bare-metal  
**Language:** Rust (no_std)  
**Bootloader:** bootloader crate (0.9+)

---

## 1. Boot Sequence

### 1.1 Bootloader Phase (handled by `bootloader` crate)
1. **BIOS/UEFI handoff** → bootloader loads kernel ELF
2. **Switch to protected mode** (32-bit) → **long mode** (64-bit)
3. **Identity map first 1 GiB** of physical memory
4. **Load GDT** with minimal 64-bit code/data segments
5. **Jump to kernel entry point** (`_start`)

### 1.2 Kernel Entry (`src/main.rs::_start`)
```
_start() {
  1. Initialize VGA text buffer (immediate feedback)
  2. Initialize GDT (Global Descriptor Table)
  3. Initialize IDT (Interrupt Descriptor Table)
  4. Initialize physical frame allocator
  5. Initialize paging (remap kernel, create new page tables)
  6. Initialize heap allocator
  7. Call kernel_main()
}
```

**Dependency order:**
- Serial UART ← none (debugging output, works before VGA)
- VGA ← none (can use immediately)
- GDT ← none (required for proper segmentation)
- IDT ← GDT (needs code segment selector)
- Frame allocator ← bootloader memory map
- Paging ← frame allocator
- Heap ← paging + frame allocator

**Panic handler:**
```rust
#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    println!("{}", info);  // VGA output
    serial_println!("{}", info);  // Serial for logging
    loop { x86_64::instructions::hlt(); }
}
```

---

## 2. Memory Layout

### 2.0 Virtual Address Space Map

```
0x0000_0000_0000_0000 ┌─────────────────────────┐
                      │ Null page (unmapped)    │
0x0000_0000_0000_1000 ├─────────────────────────┤
                      │ User space (future)     │
                      │ ...                     │
0x0000_7FFF_FFFF_FFFF ├─────────────────────────┤ ← Canonical address gap
0xFFFF_8000_0000_0000 │ (non-canonical)         │
                      │ CPU will #GP fault      │
0xFFFF_FFFF_8000_0000 ├─────────────────────────┤
                      │ Kernel code & data      │ ← Higher half (-2 GiB)
0xFFFF_FFFF_C000_0000 ├─────────────────────────┤
                      │ Physical mem offset map │ ← All physical RAM mapped here
0x4444_4444_0000      ├─────────────────────────┤
                      │ Heap (100 KiB initial)  │
0x4444_4444_0000+100K ├─────────────────────────┤
                      │ Stack (grows down)      │
                      │ ...                     │
0xFFFF_FFFF_FFFF_FFFF └─────────────────────────┘
```

**Key addresses:**
- Kernel ELF loaded by bootloader at physical ~1 MiB, mapped to `-2 GiB` virtual
- Physical memory offset mapping allows kernel to access all RAM
- Heap at fixed virtual address (allocated on demand from frame allocator)

---

## 3. Memory Management Strategy

### 3.1 Physical Frame Allocator (`src/kernel/memory/frame_allocator.rs`)

**Purpose:** Manage 4 KiB physical frames (pages)

**Design:**
- **Bitmap allocator** for simplicity:
  - 1 bit per 4 KiB frame
  - Bit = 0 → free, Bit = 1 → allocated
  - For 4 GiB RAM: 4 GiB / 4 KiB = 1,048,576 frames → 128 KiB bitmap
- Parse bootloader's memory map to mark usable regions
- Mark kernel code/data and bootloader regions as allocated

**Interface:**
```rust
pub trait FrameAllocator {
    fn allocate_frame(&mut self) -> Option<PhysFrame>;
    fn deallocate_frame(&mut self, frame: PhysFrame);
}
```

**Implementation notes:**
- Use `x86_64` crate's `PhysFrame` type
- Store bitmap in a static array (before heap is available)
- Thread-safe via Mutex (after we have heap/sync primitives)

### 3.2 Paging (`src/kernel/memory/paging.rs`)

**Purpose:** Virtual memory management via 4-level page tables

**x86_64 Paging levels:**
```
Virtual Address (48 bits used):
[47:39] = PML4 index (Level 4)
[38:30] = PDP index  (Level 3)
[29:21] = PD index   (Level 2)
[20:12] = PT index   (Level 1)
[11:0]  = Offset within 4 KiB page
```

**Initial mapping strategy:**
1. **Kernel higher-half mapping:**
   - Kernel lives in high memory: `0xFFFFFFFF80000000` and above
   - Map kernel to `-2 GiB` using 2 MiB huge pages (faster TLB)
   - Physical kernel at ~1 MiB, virtual at high addresses
   
2. **Recursive mapping technique** (or offset mapping):
   - Option A: Use bootloader's offset mapping (simple)
   - Option B: Recursive map last PML4 entry to itself (advanced)
   - Recommendation: Use bootloader's `MappedPageTable` initially

**Interface:**
```rust
pub unsafe fn init(physical_memory_offset: VirtAddr) -> OffsetPageTable<'static>;

pub fn create_mapping(
    page: Page,
    frame: PhysFrame,
    flags: PageTableFlags,
    frame_allocator: &mut impl FrameAllocator,
) -> Result<MapperFlush, MapToError>;
```

**Flags:**
- `PRESENT` - page is in memory
- `WRITABLE` - page is read/write (default read-only)
- `USER_ACCESSIBLE` - user mode can access (default kernel-only)
- `NO_EXECUTE` - NX bit (security)

### 3.3 Heap Allocator (`src/kernel/memory/heap.rs`)

**Purpose:** Dynamic memory allocation (`alloc::vec::Vec`, `Box`, etc.)

**Design:**
- Start with **fixed-size heap** (e.g., 100 KiB)
- Use `linked_list_allocator` crate (simple, audited)
- Allocate heap region from frame allocator
- Map heap region as writable pages

**Setup:**
```rust
pub const HEAP_START: usize = 0x4444_4444_0000; // arbitrary high address
pub const HEAP_SIZE: usize = 100 * 1024; // 100 KiB

#[global_allocator]
static ALLOCATOR: Locked<LinkedListAllocator> = Locked::new(LinkedListAllocator::new());

pub fn init_heap(mapper: &mut impl Mapper, frame_allocator: &mut impl FrameAllocator) 
    -> Result<(), MapToError> 
{
    // Allocate frames for heap
    // Map them as writable pages
    // Initialize ALLOCATOR
}
```

**Testing:**
- Use `alloc::boxed::Box` to verify allocation works
- Trigger OOM to verify panic handler

---

## 4. Interrupt Handling

### 4.1 IDT Setup (`src/kernel/interrupts/idt.rs`)

**Purpose:** Handle CPU exceptions and hardware interrupts

**IDT Structure:**
- 256 entries (0-255)
- Entries 0-31: CPU exceptions (divide-by-zero, page fault, etc.)
- Entries 32-255: Hardware interrupts (IRQs remapped from PIC)

**Critical exception handlers:**
```rust
extern "x86-interrupt" fn breakpoint_handler(stack_frame: InterruptStackFrame);
extern "x86-interrupt" fn double_fault_handler(stack_frame: InterruptStackFrame, error_code: u64) -> !;
extern "x86-interrupt" fn page_fault_handler(stack_frame: InterruptStackFrame, error_code: PageFaultErrorCode);
extern "x86-interrupt" fn general_protection_fault_handler(stack_frame: InterruptStackFrame, error_code: u64);
```

**Double fault safety:**
- Use separate IST (Interrupt Stack Table) entry 0
- Allocate dedicated 4 KiB stack in `.bss`:
  ```rust
  #[repr(align(4096))]
  struct Stack([u8; 4096]);
  static mut DOUBLE_FAULT_STACK: Stack = Stack([0; 4096]);
  ```
- Critical: prevents stack overflow from causing triple fault
- TSS IST[0] points to top of this stack

### 4.2 PIC Configuration (`src/kernel/interrupts/pic.rs`)

**8259 PIC (Programmable Interrupt Controller):**
- Remap IRQ 0-15 to IDT entries 32-47 (avoid conflict with CPU exceptions)
- Mask all IRQs initially except timer (IRQ 0)
- Send EOI (End of Interrupt) after handling

**Interface:**
```rust
pub fn init_pic() {
    // Remap PIC1 to 0x20, PIC2 to 0x28
    // Mask all IRQs except timer
}

pub unsafe fn notify_end_of_interrupt(irq: u8);
```

**Future:** Replace PIC with APIC (Advanced PIC) for multi-core support

### 4.3 Timer Interrupt (`src/kernel/interrupts/timer.rs`)

**Purpose:** Preemptive scheduling, uptime tracking

**Implementation:**
- Handle IRQ 0 (PIT - Programmable Interval Timer)
- Increment tick counter
- Print '.' to VGA (visual heartbeat during early development)
- Later: trigger scheduler

---

## 5. Output Drivers

### 5.1 Serial Port (UART) (`src/drivers/serial.rs`)

**Purpose:** Debugging output via COM1 (0x3F8)

**Why first:**
- Works immediately (no VGA initialization complexity)
- Output visible in QEMU via `-serial stdio`
- Essential for debugging before VGA works

**Implementation:**
```rust
use uart_16550::SerialPort;
use spin::Mutex;
use lazy_static::lazy_static;

lazy_static! {
    pub static ref SERIAL1: Mutex<SerialPort> = {
        let mut serial_port = unsafe { SerialPort::new(0x3F8) };
        serial_port.init();
        Mutex::new(serial_port)
    };
}

#[macro_export]
macro_rules! serial_print {
    ($($arg:tt)*) => {
        $crate::drivers::serial::_print(format_args!($($arg)*));
    };
}

#[macro_export]
macro_rules! serial_println {
    () => ($crate::serial_print!("\n"));
    ($fmt:expr) => ($crate::serial_print!(concat!($fmt, "\n")));
    ($fmt:expr, $($arg:tt)*) => ($crate::serial_print!(concat!($fmt, "\n"), $($arg)*));
}
```

**QEMU integration:**
```bash
qemu-system-x86_64 -drive format=raw,file=bootimage.bin -serial stdio
# Serial output appears in terminal
```

---

### 5.2 VGA Text Mode Driver

#### 5.2.1 Hardware Interface (`src/drivers/vga.rs`)

**VGA text buffer:**
- Memory-mapped I/O at physical address `0xB8000`
- 80 columns × 25 rows
- 2 bytes per character:
  - Byte 0: ASCII character
  - Byte 1: Color attribute (4 bits foreground, 4 bits background)

**Color codes:**
```rust
#[repr(u8)]
pub enum Color {
    Black = 0, Blue = 1, Green = 2, Cyan = 3,
    Red = 4, Magenta = 5, Brown = 6, LightGray = 7,
    DarkGray = 8, LightBlue = 9, LightGreen = 10, LightCyan = 11,
    LightRed = 12, Pink = 13, Yellow = 14, White = 15,
}
```

**Writer abstraction:**
```rust
pub struct Writer {
    column_position: usize,
    color_code: ColorCode,
    buffer: &'static mut Buffer,
}

impl Writer {
    pub fn write_byte(&mut self, byte: u8);
    pub fn write_string(&mut self, s: &str);
    fn new_line(&mut self);
    fn clear_row(&mut self, row: usize);
}

impl fmt::Write for Writer { /* delegate to write_string */ }
```

**Global interface:**
```rust
lazy_static! {
    pub static ref WRITER: Mutex<Writer> = Mutex::new(Writer { /* ... */ });
}

#[macro_export]
macro_rules! print {
    ($($arg:tt)*) => ($crate::drivers::vga::_print(format_args!($($arg)*)));
}

#[macro_export]
macro_rules! println {
    () => ($crate::print!("\n"));
    ($($arg:tt)*) => ($crate::print!("{}\n", format_args!($($arg)*)));
}
```

#### 5.2.2 Scrolling & Newlines

**Behavior:**
- When writing past column 79, wrap to next row
- When writing past row 24, shift all rows up (scroll)
- Clear bottom row after scroll

**SAFETY considerations:**
- VGA buffer access is `unsafe` (raw pointer dereference)
- Use `volatile` reads/writes (prevent compiler optimization)
- Wrap in safe abstraction (`Writer`)

---

## 6. Module Boundaries

```
src/
├── main.rs                    [architect]  Kernel entry, initialization sequence
│
├── kernel/
│   ├── mod.rs                 [kernel-dev] Kernel module root
│   ├── memory/
│   │   ├── mod.rs            [kernel-dev] Memory subsystem
│   │   ├── frame_allocator.rs [kernel-dev] Physical frame management
│   │   ├── paging.rs         [kernel-dev] Page table management
│   │   └── heap.rs           [kernel-dev] Heap allocator setup
│   │
│   ├── interrupts/
│   │   ├── mod.rs            [kernel-dev] Interrupt subsystem
│   │   ├── idt.rs            [kernel-dev] IDT setup & handlers
│   │   ├── pic.rs            [kernel-dev] PIC configuration
│   │   └── timer.rs          [kernel-dev] Timer interrupt handler
│   │
│   └── gdt.rs                [kernel-dev] Global Descriptor Table
│
├── drivers/
│   ├── mod.rs                [driver-dev] Driver module root
│   ├── serial.rs             [driver-dev] Serial port (UART) driver
│   └── vga.rs                [driver-dev] VGA text buffer driver
│
└── hal/                      [driver-dev] (future: abstraction layer)
    └── mod.rs

tests/
├── should_panic.rs           [reviewer]   Panic handler tests
├── vga_buffer.rs             [reviewer]   VGA driver tests
├── serial.rs                 [reviewer]   Serial driver tests
├── heap_allocation.rs        [reviewer]   Heap allocator tests
└── frame_allocator.rs        [reviewer]   Frame allocator tests
```

**Ownership rules:**
- `architect`: Designs interfaces, owns `main.rs`, `Cargo.toml`, this file
- `kernel-dev`: Implements all `src/kernel/*`
- `driver-dev`: Implements all `src/drivers/*`, `src/hal/*`
- `reviewer`: Writes tests, runs clippy, approves merges to `main`

---

## 7. Implementation Order

### Phase 1: Boot & Display (Week 1)
1. **Set up project structure** (architect)
   - `Cargo.toml` with `bootloader = "0.9"` dependency
   - `.cargo/config.toml` for x86_64 target
   - `rust-toolchain.toml` pinning nightly
   - `x86_64-unknown-none.json` target specification
   - Minimal `main.rs` with `_start` and panic handler

2. **Serial port driver** (driver-dev)
   - `src/drivers/serial.rs`
   - `serial_print!` and `serial_println!` macros
   - Test: Print "Serial: Kernel booted" visible in QEMU terminal

3. **VGA text buffer driver** (driver-dev)
   - `src/drivers/vga.rs`
   - `print!` and `println!` macros
   - Test: Print "Hello from NijOS!" in `kernel_main()`

4. **Testing harness** (reviewer)
   - Set up `cargo test` with custom test runner
   - Write unit test for VGA scrolling

**Milestone:** Can boot and print text. QEMU shows output.

---

### Phase 2: Segmentation & Interrupts (Week 2)
5. **GDT setup** (kernel-dev)
   - `src/kernel/gdt.rs`
   - Load minimal 64-bit GDT
   - Set up TSS with IST entry 0 for double fault (4096-byte stack in `.bss`)

6. **IDT & exception handling** (kernel-dev)
   - `src/kernel/interrupts/idt.rs`
   - Handlers for breakpoint, double fault, page fault
   - Test: Trigger `int3` breakpoint, verify handler runs

7. **PIC & timer** (kernel-dev)
   - `src/kernel/interrupts/pic.rs`, `timer.rs`
   - Remap PIC, enable timer interrupt
   - Print tick count every second

**Milestone:** Interrupts work. Timer ticks. System doesn't triple-fault.

---

### Phase 3: Memory Management (Week 3-4)
8. **Physical frame allocator** (kernel-dev)
   - `src/kernel/memory/frame_allocator.rs`
   - Parse bootloader memory map
   - Bitmap-based allocator

9. **Paging setup** (kernel-dev)
   - `src/kernel/memory/paging.rs`
   - Create new page tables
   - Map kernel higher-half
   - Test: Map a new page, write to it

10. **Heap allocator** (kernel-dev)
   - `src/kernel/memory/heap.rs`
   - Initialize fixed-size heap
   - Test: `Box::new()`, `Vec::push()`

**Milestone:** Dynamic memory allocation works. Can use `alloc` types.

---

### Phase 4: Stabilization & Testing (Week 5)
11. **Comprehensive testing** (reviewer)
    - Unit tests for all allocators
    - Integration tests for interrupt handling
    - Run `cargo clippy -- -D warnings`
    - Memory leak detection (if possible)

12. **Documentation** (all agents)
    - Doc comments for all public APIs
    - Update this file with lessons learned

**Milestone:** System is stable, tested, documented. Ready for next features (scheduler, syscalls, etc.).

---

## 8. Error Handling Strategy

### No Panicking in Kernel Code

**Rule:** All fallible operations return `Result<T, E>`, never `panic!` in normal paths.

**Error types:**
```rust
// Memory errors
pub enum MemoryError {
    FrameAllocationFailed,
    InvalidAddress,
    AlreadyMapped,
    NotMapped,
}

// Use Result throughout
pub fn allocate_frame() -> Result<PhysFrame, MemoryError>;
pub fn map_page(...) -> Result<MapperFlush, MapToError>;
```

**When to panic:**
- Unrecoverable initialization failures (e.g., no usable memory in bootloader map)
- Assertion violations in debug builds (`debug_assert!`)
- Explicit corruption detection (magic number mismatch, etc.)

**Panic handler behavior:**
1. Disable interrupts (`cli`)
2. Print panic info to serial (always works)
3. Print panic info to VGA (if initialized)
4. Halt CPU in loop (`hlt`)

**Future:** Add kernel debugger hooks before halt

---

## 9. Testing Strategy

### Unit Tests
- **Frame allocator:** Allocate/deallocate patterns, bitmap edge cases
- **VGA driver:** Scrolling, color codes, wrapping
- **Paging:** Map/unmap pages, flag setting

### Integration Tests
- **Boot:** System boots without triple-fault
- **Interrupts:** Breakpoint, page fault, timer all trigger correctly
- **Heap:** Large allocations, fragmentation scenarios

### Manual Tests (QEMU)
- Visual inspection of VGA output
- Timer tick rate (should be ~18.2 Hz for default PIT)
- Trigger page fault intentionally, verify handler output

---

## 10. Dependencies & Build Configuration

**Cargo.toml:**
```toml
[dependencies]
bootloader = "0.9"
x86_64 = "0.14"
uart_16550 = "0.2"
pic8259 = "0.10"
lazy_static = { version = "1.4", features = ["spin_no_std"] }
spin = "0.9"
linked_list_allocator = "0.10"

[dependencies.volatile]
version = "0.4"
```

**Build toolchain:**
```bash
rustup component add rust-src llvm-tools-preview
cargo install bootimage
```

**rust-toolchain.toml:**
```toml
[toolchain]
channel = "nightly"
components = ["rust-src", "llvm-tools-preview"]
```

**x86_64-unknown-none.json** (custom target):
```json
{
  "llvm-target": "x86_64-unknown-none",
  "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
  "arch": "x86_64",
  "target-endian": "little",
  "target-pointer-width": "64",
  "target-c-int-width": "32",
  "os": "none",
  "executables": true,
  "linker-flavor": "ld.lld",
  "linker": "rust-lld",
  "panic-strategy": "abort",
  "disable-redzone": true,
  "features": "-mmx,-sse,+soft-float"
}
```

**.cargo/config.toml:**
```toml
[build]
target = "x86_64-unknown-none.json"

[target.'cfg(target_os = "none")']
runner = "bootimage runner"
```

---

## 11. Debugging & Troubleshooting

### QEMU Debug Commands

**Basic run:**
```bash
qemu-system-x86_64 -drive format=raw,file=target/x86_64-unknown-none/debug/bootimage-nijos.bin
```

**With serial output:**
```bash
qemu-system-x86_64 \
  -drive format=raw,file=target/x86_64-unknown-none/debug/bootimage-nijos.bin \
  -serial stdio
```

**Full debugging (interrupts, CPU state):**
```bash
qemu-system-x86_64 \
  -drive format=raw,file=target/x86_64-unknown-none/debug/bootimage-nijos.bin \
  -serial stdio \
  -d int,cpu_reset \
  -no-reboot \
  -no-shutdown
```

**GDB debugging:**
```bash
# Terminal 1:
qemu-system-x86_64 -drive format=raw,file=bootimage.bin -s -S

# Terminal 2:
rust-gdb target/x86_64-unknown-none/debug/nijos
(gdb) target remote :1234
(gdb) break _start
(gdb) continue
```

---

### Common Issues

**Triple fault on boot:**
- Check GDT is loaded before IDT
- Verify TSS has valid IST entry for double fault
- Use QEMU `-d int` flag to see interrupt log

**Page fault loop:**
- Ensure page tables are mapped correctly
- Check all accessed addresses are within mapped regions
- Use recursive/offset mapping to access page tables themselves

**Heap allocation fails:**
- Verify heap region is mapped as WRITABLE
- Check `linked_list_allocator` is initialized with correct size
- Ensure frame allocator has free frames

**QEMU hangs:**
- Likely infinite loop or interrupt not EOI'd
- Check timer handler sends EOI to PIC
- Use QEMU monitor (`Ctrl-A C`) to inspect state

---

## 12. Future Work (Post-MVP)

- **Keyboard driver:** PS/2 keyboard via IRQ 1
- **APIC/APIC:** Replace PIC for SMP (multi-core) support
- **Scheduler:** Round-robin task switching
- **Syscalls:** User mode → kernel mode transitions
- **Filesystem:** Read-only FAT32 or custom FS
- **Network stack:** E1000 NIC driver + basic TCP/IP

---

## References

- [OSDev Wiki](https://wiki.osdev.org/) - x86_64, paging, interrupts
- [Philipp Oppermann's Blog](https://os.phil-opp.com/) - Rust OS tutorial (excellent)
- [x86_64 crate docs](https://docs.rs/x86_64/) - Rust abstractions for x86_64
- [Intel SDM](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) - Authoritative x86_64 reference

---

**Document Owner:** architect  
**Last Updated:** 2026-03-20  
**Status:** Initial design - ready for implementation
