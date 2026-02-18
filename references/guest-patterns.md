# Guest Translation Patterns

## Entry Point Rewriting

### R0VM → SP1
```rust
// FROM (R0VM):
#![no_main]
risc0_zkvm::guest::entry!(main);
fn main() { /* ... */ }

// TO (SP1):
#![no_main]
sp1_zkvm::entrypoint!(main);
fn main() { /* ... */ }
```

### R0VM → OpenVM
```rust
// FROM (R0VM):
#![no_main]
risc0_zkvm::guest::entry!(main);
fn main() { /* ... */ }

// TO (OpenVM):
// No #![no_main] needed
#[openvm::main]
fn main() { /* ... */ }
```

### R0VM → Zisk
```rust
// FROM (R0VM):
#![no_main]
risc0_zkvm::guest::entry!(main);
fn main() { /* ... */ }

// TO (Zisk):
#![no_main]
#![no_std]
extern crate alloc;
fn main() { /* ... */ }
```

### R0VM → Jolt
```rust
// FROM (R0VM):
#![no_main]
risc0_zkvm::guest::entry!(main);
fn main() {
    let input: MyInput = env::read();
    let result = compute(input);
    env::commit(&result);
}

// TO (Jolt):
// See references/jolt-special.md for full restructuring guide
#[jolt::provable]
fn compute(input: MyInput) -> MyResult {
    // Pure computation, no I/O calls
    compute(input)
}
```

### SP1 → R0VM
```rust
// FROM (SP1):
#![no_main]
sp1_zkvm::entrypoint!(main);
fn main() { /* ... */ }

// TO (R0VM):
#![no_main]
risc0_zkvm::guest::entry!(main);
fn main() { /* ... */ }
```

### SP1 → OpenVM
```rust
// FROM (SP1):
#![no_main]
sp1_zkvm::entrypoint!(main);
fn main() { /* ... */ }

// TO (OpenVM):
#[openvm::main]
fn main() { /* ... */ }
```

### OpenVM → R0VM
```rust
// FROM (OpenVM):
#[openvm::main]
fn main() { /* ... */ }

// TO (R0VM):
#![no_main]
risc0_zkvm::guest::entry!(main);
fn main() { /* ... */ }
```

### OpenVM → SP1
```rust
// FROM (OpenVM):
#[openvm::main]
fn main() { /* ... */ }

// TO (SP1):
#![no_main]
sp1_zkvm::entrypoint!(main);
fn main() { /* ... */ }
```

### Zisk → R0VM
```rust
// FROM (Zisk):
#![no_main]
#![no_std]
extern crate alloc;
fn main() { /* ... */ }

// TO (R0VM):
#![no_main]
// Remove #![no_std] — R0VM supports std
risc0_zkvm::guest::entry!(main);
fn main() { /* ... */ }
```

### Zisk → SP1
```rust
// FROM (Zisk):
#![no_main]
#![no_std]
extern crate alloc;
fn main() { /* ... */ }

// TO (SP1):
#![no_main]
// Remove #![no_std] — SP1 supports std
sp1_zkvm::entrypoint!(main);
fn main() { /* ... */ }
```

## Structured I/O — Read Substitutions

### R0VM → SP1
```rust
// FROM:
use risc0_zkvm::guest::env;
let val: T = env::read();

// TO:
let val: T = sp1_zkvm::io::read::<T>();
```

### R0VM → OpenVM
```rust
// FROM:
use risc0_zkvm::guest::env;
let val: T = env::read();

// TO:
let val: T = openvm::io::read::<T>();
```

### R0VM → Zisk
```rust
// FROM:
use risc0_zkvm::guest::env;
let val: T = env::read();

// TO:
let val: T = ziskos::read::<T>();
```

### SP1 → R0VM
```rust
// FROM:
let val: T = sp1_zkvm::io::read::<T>();

// TO:
use risc0_zkvm::guest::env;
let val: T = env::read();
```

## Structured I/O — Commit Substitutions

### R0VM → SP1
```rust
// FROM:
env::commit(&result);
// TO:
sp1_zkvm::io::commit(&result);
```

### R0VM → OpenVM
```rust
// FROM:
env::commit(&result);
// TO:
openvm::io::reveal(&result, 0);
// TRANSFER NOTE: Second argument is the reveal channel (0 = default public output)
```

### R0VM → Zisk
```rust
// FROM:
env::commit(&result);
// TO:
ziskos::commit(&result);
```

### SP1 → R0VM
```rust
// FROM:
sp1_zkvm::io::commit(&result);
// TO:
env::commit(&result);
```

### SP1 → OpenVM
```rust
// FROM:
sp1_zkvm::io::commit(&result);
// TO:
openvm::io::reveal(&result, 0);
```

## Raw I/O — Byte Reading Helpers

### R0VM Raw Read Pattern
R0VM reads are u32-word-aligned. Common pattern for reading typed data from raw bytes:

```rust
// R0VM: Reading raw bytes via u32-aligned reads
fn read_bytes_raw(n: usize) -> Vec<u8> {
    let n_words = (n + 3) / 4;
    let mut words = vec![0u32; n_words];
    risc0_zkvm::guest::env::read_slice(&mut words);
    let bytes: Vec<u8> = words.iter()
        .flat_map(|w| w.to_le_bytes())
        .take(n)
        .collect();
    bytes
}

fn read_i8_raw() -> i8 {
    let bytes = read_bytes_raw(1);
    bytes[0] as i8
}

fn read_i32_raw() -> i32 {
    let bytes = read_bytes_raw(4);
    i32::from_le_bytes([bytes[0], bytes[1], bytes[2], bytes[3]])
}
```

### SP1 Raw Read Pattern
SP1 returns `Vec<u8>` directly — simpler:

```rust
// SP1: Reading raw bytes
fn read_bytes_raw(n: usize) -> Vec<u8> {
    let data = sp1_zkvm::io::read_vec();
    // TRANSFER NOTE: SP1 read_vec() returns entire submitted slice.
    // Caller should verify length matches expected.
    assert_eq!(data.len(), n, "unexpected input length");
    data
}

fn read_i8_raw() -> i8 {
    let data = sp1_zkvm::io::read_vec();
    data[0] as i8
}

fn read_i32_raw() -> i32 {
    let data = sp1_zkvm::io::read_vec();
    i32::from_le_bytes([data[0], data[1], data[2], data[3]])
}
```

### OpenVM Raw Read Pattern
```rust
// OpenVM: Reading raw bytes
fn read_bytes_raw(_n: usize) -> Vec<u8> {
    openvm::io::read_vec()
}

fn read_i8_raw() -> i8 {
    let data = openvm::io::read_vec();
    data[0] as i8
}

fn read_i32_raw() -> i32 {
    let data = openvm::io::read_vec();
    i32::from_le_bytes([data[0], data[1], data[2], data[3]])
}
```

### Zisk Raw Read Pattern
```rust
// Zisk: Reading raw bytes (no_std)
fn read_bytes_raw(_n: usize) -> alloc::vec::Vec<u8> {
    ziskos::read_slice()
}

fn read_i8_raw() -> i8 {
    let data = ziskos::read_slice();
    data[0] as i8
}

fn read_i32_raw() -> i32 {
    let data = ziskos::read_slice();
    i32::from_le_bytes([data[0], data[1], data[2], data[3]])
}
```

### Converting R0VM Raw Reads to SP1/OpenVM

When the source code has multiple sequential `env::read_slice()` calls reading different chunks, each one maps to a separate `read_vec()` call on the target. The host must submit inputs in the same order.

```rust
// R0VM source — reading weight chunks:
let mut chunk1 = vec![0u32; chunk1_words];
env::read_slice(&mut chunk1);
let mut chunk2 = vec![0u32; chunk2_words];
env::read_slice(&mut chunk2);

// SP1 target:
let chunk1_bytes = sp1_zkvm::io::read_vec();
let chunk2_bytes = sp1_zkvm::io::read_vec();
// TRANSFER NOTE: Byte order preserved. No u32 alignment padding needed on SP1.
```

## Raw I/O — Write/Commit Bytes

### R0VM → SP1
```rust
// FROM (R0VM):
env::commit_slice(&output_words); // &[u32]

// TO (SP1):
// TRANSFER NOTE: SP1 commit_slice takes &[u8], not &[u32].
// Convert u32 words to bytes if needed:
let output_bytes: Vec<u8> = output_words.iter()
    .flat_map(|w| w.to_le_bytes())
    .collect();
sp1_zkvm::io::commit_slice(&output_bytes);
```

### R0VM → OpenVM
```rust
// FROM (R0VM):
env::commit_slice(&output_words);

// TO (OpenVM):
let output_bytes: Vec<u8> = output_words.iter()
    .flat_map(|w| w.to_le_bytes())
    .collect();
openvm::io::reveal_slice(&output_bytes, 0);
```

## Cycle Counting

### R0VM → SP1
```rust
// FROM:
let before = risc0_zkvm::guest::env::cycle_count();
// ... computation ...
let after = risc0_zkvm::guest::env::cycle_count();
eprintln!("cycles: {}", after - before);

// TO:
let before = sp1_zkvm::io::hint_cycle_count();
// ... computation ...
let after = sp1_zkvm::io::hint_cycle_count();
// TRANSFER NOTE: SP1 cycle count is a hint (not proven). Value may differ from R0VM.
eprintln!("cycles: {}", after - before);
```

### R0VM → OpenVM / Zisk / Jolt
```rust
// FROM:
let cycles = risc0_zkvm::guest::env::cycle_count();

// TO:
let cycles = 0u64;
// TRANSFER NOTE: Cycle counting not available on this platform. Stubbed to 0.
```

## SP1 Hint Read Distinction

SP1 has separate APIs for proven reads vs. hint (unproven) reads. When transferring FROM a platform where all reads use the same API:

```rust
// If the original code reads data that the host provides as hints
// (not part of the public input/output), use SP1's hint API:

// Proven public input:
let public_input: T = sp1_zkvm::io::read::<T>();

// Unproven hint from host (e.g., precomputed results for Freivalds verification):
let hint_data: T = sp1_zkvm::io::hint_read::<T>();
let hint_bytes: Vec<u8> = sp1_zkvm::io::hint_read_vec();
```

**TRANSFER NOTE**: On the host side, SP1 distinguishes between `stdin.write()` (proven) and `stdin.write()` for hints. The order of reads in the guest must match the order of writes in the host.

## Allocator Setup

### Platforms with `std` (R0VM, SP1, OpenVM)
No special allocator setup needed. Standard `Vec`, `String`, etc. work out of the box.

### R0VM with `heap-embedded-alloc`
If the source uses a custom heap size:
```rust
// R0VM guest with custom heap:
risc0_zkvm::guest::entry!(main);
// Heap is configured via risc0.toml or linker script
```

### Zisk (`no_std`)
```rust
#![no_std]
#![no_main]
extern crate alloc;
// Zisk provides a global allocator automatically
// Vec, String, etc. available via `alloc` crate
use alloc::vec::Vec;
use alloc::string::String;
```

### Jolt
```rust
// Jolt guest is a library — std is available
// No special allocator setup needed
#[jolt::provable]
fn compute(input: Vec<u8>) -> Vec<u8> {
    // std collections work normally
}
```

## Real-World Transfer Lessons (from RSP SP1 → R0VM)

### Environment Variables in Guest
R0VM disables `sys_getenv` by default. If the guest code (or any dependency like rayon)
reads environment variables, you MUST:
1. Add `risc0-zkvm-platform = { version = "2.2", features = ["sys-getenv"] }` to guest Cargo.toml
2. Pass env vars from host: `.env_var("RAYON_NUM_THREADS", "1")` on ExecutorEnv builder

### SP1-Specific BLS12-381 (kzg-rs)
Succinct's `kzg-rs` depends on `sp1_bls12_381` which uses SP1 syscalls. When transferring
to R0VM, you need a local fork replacing `sp1_bls12_381` with standard `bls12_381` (patched
to R0VM's accelerated version). Also need to implement `msm_variable_base` as a standalone
function since the standard crate doesn't have it.

### Patch Sections for Git Dependencies
If the source project depends on crates from git (e.g., `kzg-rs` from `https://github.com/succinctlabs/kzg-rs`),
you need BOTH:
- `[patch.crates-io]` for crates.io dependencies
- `[patch."https://github.com/succinctlabs/kzg-rs"]` for git dependencies

### Guest Workspace Detachment
R0VM guest crate needs BOTH:
- Empty `[workspace]` table in guest Cargo.toml
- `exclude = ["methods/guest"]` in workspace root Cargo.toml

### Error Type Bridging
R0VM SDK returns `anyhow::Error`, not `eyre`. Use `.map_err(|e| eyre::eyre!("{e}"))` pattern.

## Common Import Block Translations

### R0VM Guest Imports
```rust
use risc0_zkvm::guest::env;
```

### SP1 Guest Imports
```rust
// No single import — use full paths:
// sp1_zkvm::io::read::<T>()
// sp1_zkvm::io::commit(&val)
// sp1_zkvm::io::read_vec()
// sp1_zkvm::io::hint_read::<T>()
```

### OpenVM Guest Imports
```rust
use openvm::io::{read, reveal, read_vec, reveal_slice};
```

### Zisk Guest Imports
```rust
use ziskos::{read, commit, read_slice, write};
```
