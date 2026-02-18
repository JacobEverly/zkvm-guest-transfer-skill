# Cross-Platform zkVM API Mapping

## Entry Points

| Platform | Entry Point | Attributes Required |
|----------|------------|-------------------|
| **R0VM** | `risc0_zkvm::guest::entry!(main)` | `#![no_main]` |
| **SP1** | `sp1_zkvm::entrypoint!(main)` | `#![no_main]` |
| **OpenVM** | `#[openvm::main] fn main()` | None (no `#![no_main]`) |
| **Zisk** | Bare `fn main()` | `#![no_main]`, `#![no_std]` |
| **Jolt** | `#[jolt::provable] fn compute(args) -> ret` | None (library, not binary) |

## Structured I/O — Read (Serde-deserialized)

| Platform | API | Notes |
|----------|-----|-------|
| **R0VM** | `risc0_zkvm::guest::env::read::<T>()` | T: Deserialize |
| **SP1** | `sp1_zkvm::io::read::<T>()` | T: Deserialize |
| **OpenVM** | `openvm::io::read::<T>()` | T: Deserialize |
| **Zisk** | `ziskos::read::<T>()` | T: Deserialize |
| **Jolt** | Function parameters | Passed directly by prover |

## Structured I/O — Commit (Public output)

| Platform | API | Notes |
|----------|-----|-------|
| **R0VM** | `risc0_zkvm::guest::env::commit(&val)` | Serializes to journal |
| **SP1** | `sp1_zkvm::io::commit(&val)` | Serializes to public values |
| **OpenVM** | `openvm::io::reveal(&val, 0)` | Second arg is channel (usually 0) |
| **Zisk** | `ziskos::commit(&val)` | Serializes to public output |
| **Jolt** | Return value | Returned from provable function |

## Raw I/O — Read Bytes

| Platform | API | Alignment | Notes |
|----------|-----|-----------|-------|
| **R0VM** | `env::read_slice(&mut buf)` | u32-word-aligned | Reads into pre-allocated `&mut [u32]` or via `env::read_slice` for bytes |
| **SP1** | `sp1_zkvm::io::read_vec()` | Byte-aligned | Returns `Vec<u8>`, no pre-allocation needed |
| **OpenVM** | `openvm::io::read_vec()` | Byte-aligned | Returns `Vec<u8>` |
| **Zisk** | `ziskos::read_slice()` | Platform-specific | Returns byte slice |
| **Jolt** | N/A | — | No raw I/O; pass as function arg (e.g., `Vec<u8>`) |

**IMPORTANT**: When transferring raw I/O from R0VM to SP1/OpenVM, the u32-alignment constraint is removed. Helper functions like `read_i8_raw`, `read_i32_raw` that manually reconstruct from u32 words can be simplified to direct byte reads on the target.

## Raw I/O — Write/Commit Bytes

| Platform | API | Notes |
|----------|-----|-------|
| **R0VM** | `env::commit_slice(&[u32])` | u32-word-aligned output |
| **SP1** | `sp1_zkvm::io::commit_slice(&[u8])` | Byte slice output |
| **OpenVM** | `openvm::io::reveal_slice(&[u8], 0)` | Channel 0 for public output |
| **Zisk** | `ziskos::write(&[u8])` | Byte slice output |
| **Jolt** | Return value | Bundle into return struct |

## Hints / Advice (Host → Guest, non-proven)

| Platform | Guest Read | Host Write | Notes |
|----------|-----------|------------|-------|
| **R0VM** | `env::read()` / `env::read_slice()` | `env_builder.write()` / `.write_slice()` | Same channel as proven I/O; host decides what to send |
| **SP1** | `sp1_zkvm::io::hint_read::<T>()` / `hint_read_vec()` | `stdin.write()` / `.write_slice()` | Separate hint channel |
| **OpenVM** | `openvm::io::read_vec()` | Via stdin setup | No dedicated hint API |
| **Zisk** | `ziskos::read()` | Via input setup | No dedicated hint API |
| **Jolt** | N/A | N/A | No hint mechanism |

**TRANSFER NOTE**: SP1 distinguishes between proven reads (`io::read`) and hint reads (`io::hint_read`). When transferring FROM R0VM where all reads use the same API, you must determine which reads are for proven public inputs vs. unproven hints and use the appropriate SP1 API.

## Cycle Counting / Profiling

| Platform | API | Returns |
|----------|-----|---------|
| **R0VM** | `risc0_zkvm::guest::env::cycle_count()` | `u64` |
| **SP1** | `sp1_zkvm::io::hint_cycle_count()` | `u64` (hint only, not proven) |
| **OpenVM** | No equivalent | Stub → `0u64` |
| **Zisk** | No equivalent | Stub → `0u64` |
| **Jolt** | No equivalent | Stub → `0u64` |

## Guest Crate Dependencies

| Platform | Crate | Version (as of 2025) | Features |
|----------|-------|---------------------|----------|
| **R0VM** | `risc0-zkvm` | `^3.0` | `guest` feature for guest code |
| **SP1** | `sp1-zkvm` | `4.0` | Default features for guest |
| **OpenVM** | `openvm` | `1.0` | Default features |
| **Zisk** | `ziskos` | `0.1` | — |
| **Jolt** | `jolt-sdk` | `0.1` | — |

## Target Triples

| Platform | Target | Notes |
|----------|--------|-------|
| **R0VM** | `riscv32im-risc0-zkvm-elf` | RISC-V 32-bit, M extension |
| **SP1** | `riscv32im-succinct-zkvm-elf` | RISC-V 32-bit, M extension |
| **OpenVM** | Custom RISC-V with extensions | Platform-specific extensions |
| **Zisk** | Custom RISC-V | Zisk-specific target |
| **Jolt** | `riscv32i-unknown-none-elf` | Base RISC-V 32-bit (no M extension) |

## Precompiles / Accelerated Operations

| Operation | R0VM | SP1 | OpenVM | Zisk | Jolt |
|-----------|------|-----|--------|------|------|
| SHA-256 | `risc0-zkvm` (accelerated) | `sp1-zkvm` patch (accelerated) | Via extensions | Software only | Software only |
| Keccak-256 | Software / accelerated | `sp1-zkvm` patch (accelerated) | Via extensions | Software only | Software only |
| BigInt/RSA | `risc0-bigint2` | `sp1-bigint` | Via extensions | Software only | Software only |
| Ed25519 | Via crate patches | Via crate patches | Via extensions | Software only | Software only |
| Secp256k1 | Via crate patches | Via crate patches | Via extensions | Software only | Software only |

**TRANSFER NOTE**: When transferring code that relies on accelerated precompiles (e.g., SHA-256 on R0VM) to a platform without them (e.g., Jolt), the code will still work but proving will be significantly slower since the operations run in software on the base ISA.
