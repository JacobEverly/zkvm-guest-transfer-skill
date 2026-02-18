# Build Systems Reference

## Toolchain Installation

| Platform | Install Command | What It Provides |
|----------|----------------|-----------------|
| **R0VM** | `curl -L https://risczero.com/install \| bash && rzup` | `cargo risczero`, RISC-V toolchain, prover |
| **SP1** | `curl -L https://sp1.succinct.xyz \| bash && sp1up` | `cargo prove`, SP1 toolchain, prover |
| **OpenVM** | `cargo install openvm-cli` | `cargo openvm`, OpenVM toolchain |
| **Zisk** | `curl -L https://zisk.io/install \| bash && ziskup` | `cargo zisk`, Zisk toolchain |
| **Jolt** | `cargo install jolt-cli` (or add as dependency) | Jolt SDK, no special toolchain |

## Workspace Layout

### R0VM Workspace
```
my-project/
  Cargo.toml          # Workspace root
  methods/
    guest/
      Cargo.toml      # Guest crate
      src/main.rs
    build.rs           # risc0_build::embed_methods()
    Cargo.toml         # Methods crate (build-dep on risc0-build)
  host/
    Cargo.toml         # Host crate (dep on methods + risc0-zkvm)
    src/main.rs
```

**Workspace Cargo.toml:**
```toml
[workspace]
members = ["host", "methods"]
resolver = "2"
```

**methods/Cargo.toml:**
```toml
[package]
name = "my-project-methods"
version = "0.1.0"
edition = "2021"

[build-dependencies]
risc0-build = "3.0"
```

**methods/build.rs:**
```rust
fn main() {
    risc0_build::embed_methods();
}
```

**methods/guest/Cargo.toml:**
```toml
[package]
name = "my-project-guest"
version = "0.1.0"
edition = "2021"

[dependencies]
risc0-zkvm = { version = "3.0", default-features = false, features = ["guest"] }
serde = { version = "1.0", features = ["derive"] }
```

### SP1 Workspace
```
my-project/
  Cargo.toml          # Workspace root
  program/
    Cargo.toml         # Guest crate
    src/main.rs
  script/
    Cargo.toml         # Host crate
    src/main.rs
    build.rs           # sp1_build::build_program()
```

**Workspace Cargo.toml:**
```toml
[workspace]
members = ["program", "script"]
resolver = "2"
```

**program/Cargo.toml:**
```toml
[package]
name = "my-project-program"
version = "0.1.0"
edition = "2021"

[dependencies]
sp1-zkvm = "4.0"
serde = { version = "1.0", features = ["derive"] }
```

**script/Cargo.toml:**
```toml
[package]
name = "my-project-script"
version = "0.1.0"
edition = "2021"

[dependencies]
sp1-sdk = "4.0"
serde = { version = "1.0", features = ["derive"] }

[build-dependencies]
sp1-build = "4.0"
```

**script/build.rs:**
```rust
fn main() {
    sp1_build::build_program("../program");
}
```

### OpenVM Workspace
```
my-project/
  Cargo.toml          # Workspace root (optional)
  guest/
    Cargo.toml
    src/main.rs
  host/
    Cargo.toml
    src/main.rs
```

**guest/Cargo.toml:**
```toml
[package]
name = "my-project-guest"
version = "0.1.0"
edition = "2021"

[dependencies]
openvm = "1.0"
serde = { version = "1.0", features = ["derive"] }
```

### Zisk Workspace
```
my-project/
  Cargo.toml          # Workspace root (optional)
  guest/
    Cargo.toml
    src/main.rs
  host/
    Cargo.toml
    src/main.rs
```

**guest/Cargo.toml:**
```toml
[package]
name = "my-project-guest"
version = "0.1.0"
edition = "2021"

[dependencies]
ziskos = "0.1"
serde = { version = "1.0", features = ["derive", "alloc"] }  # alloc for no_std
```

**Note:** Zisk guests are `#![no_std]` — use `serde` with `alloc` feature instead of `std`.

### Jolt Project
```
my-project/
  Cargo.toml          # Workspace root
  guest/
    Cargo.toml         # Library crate (not binary!)
    src/lib.rs          # #[jolt::provable] functions
  host/
    Cargo.toml
    src/main.rs
```

**guest/Cargo.toml:**
```toml
[package]
name = "my-project-guest"
version = "0.1.0"
edition = "2021"

[dependencies]
jolt-sdk = "0.1"
serde = { version = "1.0", features = ["derive"] }
```

**Note:** Jolt guest is a **library** (`lib.rs`), not a binary (`main.rs`).

## Build Commands

| Platform | Build Guest | Build Host | Run (Execute Only) | Generate Proof |
|----------|------------|------------|-------------------|---------------|
| **R0VM** | `cargo build` (workspace builds guest via methods/build.rs) | Same workspace build | `RISC0_DEV_MODE=1 cargo run --release` | `cargo run --release` |
| **SP1** | `cargo prove build` in program/ | `cargo build` in script/ | `SP1_PROVER=mock cargo run --release` | `cargo run --release` |
| **OpenVM** | `cargo openvm build` in guest/ | `cargo build` in host/ | `cargo openvm run` | `cargo openvm prove` |
| **Zisk** | `cargo zisk build` in guest/ | `cargo build` in host/ | `cargo zisk run` | `cargo zisk prove` |
| **Jolt** | `cargo build` (library crate) | `cargo build` in host/ | `cargo run` | `cargo run --release` |

## Build Environment Variables

| Platform | Variable | Purpose |
|----------|----------|---------|
| **R0VM** | `RISC0_DEV_MODE=1` | Skip proving, execute only (fast testing) |
| **R0VM** | `RISC0_PPROF_OUT=profile.pb` | Enable profiling |
| **SP1** | `SP1_PROVER=mock` | Use mock prover (fast testing) |
| **SP1** | `SP1_PROVER=network` | Use Succinct's prover network |
| **SP1** | `RUST_LOG=info` | Enable SP1 logging |

## Guest `.cargo/config.toml`

### R0VM
```toml
[build]
target = "riscv32im-risc0-zkvm-elf"

[unstable]
build-std = ["core", "alloc", "std"]
build-std-features = ["compiler-builtins-mem"]
```

### SP1
```toml
[build]
target = "riscv32im-succinct-zkvm-elf"

[unstable]
build-std = ["core", "alloc", "std"]
build-std-features = ["compiler-builtins-mem"]
```

### Jolt
```toml
[build]
target = "riscv32i-unknown-none-elf"

[unstable]
build-std = ["core", "alloc"]
build-std-features = ["compiler-builtins-mem"]
```

## Patch Sections for Precompiles

### R0VM SHA-256 Acceleration
```toml
# In workspace Cargo.toml
[patch.crates-io]
sha2 = { git = "https://github.com/risc0/RustCrypto-hashes", tag = "sha2-v0.10.8-risczero.1" }
```

### SP1 SHA-256 / Keccak Acceleration
```toml
# In workspace Cargo.toml
[patch.crates-io]
sha2 = { git = "https://github.com/sp1-patches/RustCrypto-hashes", branch = "patch-sha2-v0.10.8" }
tiny-keccak = { git = "https://github.com/sp1-patches/tiny-keccak", branch = "patch-v2.0.2" }
```

**TRANSFER NOTE**: When transferring between R0VM and SP1, update the `[patch.crates-io]` section to use the target platform's patched crates. If transferring to OpenVM/Zisk/Jolt, remove these patches (crypto operations run in software).
