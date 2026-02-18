# Host / Prover SDK Patterns

## R0VM Host Template

### Cargo.toml
```toml
[package]
name = "my-project-host"
version = "0.1.0"
edition = "2021"

[dependencies]
risc0-zkvm = "3.0"
serde = { version = "1.0", features = ["derive"] }
bincode = "1.3"   # If using custom serialization
```

### build.rs
```rust
fn main() {
    risc0_build::embed_methods();
}
```

### src/main.rs — Structured I/O
```rust
use risc0_zkvm::{default_prover, ExecutorEnv};

// Include the compiled guest ELF
include!(concat!(env!("OUT_DIR"), "/methods.rs"));

fn main() {
    // Build executor environment with inputs
    let env = ExecutorEnv::builder()
        .write(&my_input)       // Serde-serialized structured input
        .unwrap()
        .build()
        .unwrap();

    // Generate proof
    let prover = default_prover();
    let prove_info = prover
        .prove(env, MY_GUEST_ELF)
        .unwrap();

    let receipt = prove_info.receipt;

    // Verify the proof
    receipt.verify(MY_GUEST_ID).unwrap();

    // Extract public output from journal
    let output: MyOutput = receipt.journal.decode().unwrap();
    println!("Verified output: {:?}", output);
}
```

### src/main.rs — Raw Slice I/O
```rust
use risc0_zkvm::{default_prover, ExecutorEnv};

include!(concat!(env!("OUT_DIR"), "/methods.rs"));

fn main() {
    // For raw bytes, convert to u32 words (R0VM requirement)
    let raw_data: Vec<u8> = load_data();
    let padded_len = (raw_data.len() + 3) / 4 * 4;
    let mut padded = raw_data.clone();
    padded.resize(padded_len, 0);
    let words: Vec<u32> = padded.chunks(4)
        .map(|c| u32::from_le_bytes([c[0], c[1], c[2], c[3]]))
        .collect();

    let env = ExecutorEnv::builder()
        .write_slice(&words)
        .build()
        .unwrap();

    let prover = default_prover();
    let prove_info = prover.prove(env, MY_GUEST_ELF).unwrap();
    let receipt = prove_info.receipt;
    receipt.verify(MY_GUEST_ID).unwrap();

    // Read raw journal bytes
    let journal_bytes = receipt.journal.bytes.clone();
    println!("Journal ({} bytes): {:?}", journal_bytes.len(), &journal_bytes[..20]);
}
```

### src/main.rs — Weight Streaming Pattern (Large Models)
```rust
use risc0_zkvm::{default_prover, ExecutorEnv};
use std::fs;

include!(concat!(env!("OUT_DIR"), "/methods.rs"));

fn main() {
    let mut env_builder = ExecutorEnv::builder();

    // Write structured config first
    env_builder.write(&config).unwrap();

    // Stream weight files as raw slices
    for weight_file in &weight_files {
        let data = fs::read(weight_file).unwrap();
        let words = bytes_to_u32_words(&data);
        env_builder.write_slice(&words);
    }

    let env = env_builder.build().unwrap();
    let prover = default_prover();
    let prove_info = prover.prove(env, MY_GUEST_ELF).unwrap();
    // ...
}

fn bytes_to_u32_words(data: &[u8]) -> Vec<u32> {
    let padded_len = (data.len() + 3) / 4 * 4;
    let mut padded = data.to_vec();
    padded.resize(padded_len, 0);
    padded.chunks(4)
        .map(|c| u32::from_le_bytes([c[0], c[1], c[2], c[3]]))
        .collect()
}
```

---

## SP1 Host Template

### Cargo.toml
```toml
[package]
name = "my-project-host"
version = "0.1.0"
edition = "2021"

[dependencies]
sp1-sdk = "4.0"
serde = { version = "1.0", features = ["derive"] }
```

### build.rs
```rust
fn main() {
    sp1_build::build_program("../guest");
}
```

### src/main.rs — Structured I/O
```rust
use sp1_sdk::{ProverClient, SP1Stdin};

// The ELF is embedded at build time
const GUEST_ELF: &[u8] = include_bytes!("../../guest/elf/riscv32im-succinct-zkvm-elf");

fn main() {
    // Setup prover client
    let client = ProverClient::from_env();

    // Prepare stdin with inputs
    let mut stdin = SP1Stdin::new();
    stdin.write(&my_input);  // Serde-serialized

    // Generate proof
    let (pk, vk) = client.setup(GUEST_ELF);
    let proof = client.prove(&pk, &stdin).compressed().run().unwrap();

    // Verify
    client.verify(&proof, &vk).unwrap();

    // Extract public values
    let output: MyOutput = proof.public_values.read();
    println!("Verified output: {:?}", output);
}
```

### src/main.rs — Raw Slice I/O
```rust
use sp1_sdk::{ProverClient, SP1Stdin};

const GUEST_ELF: &[u8] = include_bytes!("../../guest/elf/riscv32im-succinct-zkvm-elf");

fn main() {
    let client = ProverClient::from_env();
    let mut stdin = SP1Stdin::new();

    // Raw bytes — no u32 alignment needed on SP1
    let raw_data: Vec<u8> = load_data();
    stdin.write_slice(&raw_data);

    let (pk, vk) = client.setup(GUEST_ELF);
    let proof = client.prove(&pk, &stdin).compressed().run().unwrap();
    client.verify(&proof, &vk).unwrap();

    // Read raw public values
    let public_bytes = proof.public_values.as_slice();
    println!("Public values ({} bytes)", public_bytes.len());
}
```

### src/main.rs — Weight Streaming Pattern
```rust
use sp1_sdk::{ProverClient, SP1Stdin};
use std::fs;

const GUEST_ELF: &[u8] = include_bytes!("../../guest/elf/riscv32im-succinct-zkvm-elf");

fn main() {
    let client = ProverClient::from_env();
    let mut stdin = SP1Stdin::new();

    // Structured config
    stdin.write(&config);

    // Stream weight files as raw byte slices
    // TRANSFER NOTE: No u32 alignment needed on SP1. Write bytes directly.
    for weight_file in &weight_files {
        let data = fs::read(weight_file).unwrap();
        stdin.write_slice(&data);
    }

    let (pk, vk) = client.setup(GUEST_ELF);
    let proof = client.prove(&pk, &stdin).compressed().run().unwrap();
    client.verify(&proof, &vk).unwrap();
}
```

### SP1 Execute-Only Mode (No Proof)
```rust
// Useful for testing transfers before generating proofs
let client = ProverClient::from_env();
let mut stdin = SP1Stdin::new();
stdin.write(&my_input);

let (output, _report) = client.execute(GUEST_ELF, &stdin).run().unwrap();
let result: MyOutput = output.read();
println!("Execution result (no proof): {:?}", result);
```

---

## OpenVM Host Template

### Cargo.toml
```toml
[package]
name = "my-project-host"
version = "0.1.0"
edition = "2021"

[dependencies]
openvm-sdk = "1.0"
serde = { version = "1.0", features = ["derive"] }
```

### CLI-Based Proving (Recommended for Simple Cases)
```bash
# Build the guest program
cargo openvm build --manifest-path guest/Cargo.toml

# Generate proof
cargo openvm prove --manifest-path guest/Cargo.toml --input input.bin

# Verify proof
cargo openvm verify --proof proof.bin
```

### src/main.rs — SDK-Based Proving
```rust
use openvm_sdk::{Sdk, StdIn};

fn main() {
    let sdk = Sdk::new();

    // Load compiled guest program
    let program = sdk.load_program("./guest/target/openvm/guest.vmexe").unwrap();

    // Prepare inputs
    let mut stdin = StdIn::new();
    stdin.write(&my_input);         // Serde-serialized
    // stdin.write_bytes(&raw_data); // Raw bytes

    // Execute (no proof, for testing)
    let output = sdk.execute(program.clone(), stdin.clone()).unwrap();

    // Generate proof
    let proof = sdk.prove(program, stdin).unwrap();

    // Verify
    sdk.verify(&proof).unwrap();

    println!("Proof verified successfully");
}
```

**TRANSFER NOTE**: OpenVM's SDK API is evolving. The CLI-based approach (`cargo openvm prove`) is more stable for initial transfers. Check OpenVM documentation for the latest SDK API.

---

## Zisk Host Template

### Cargo.toml
```toml
[package]
name = "my-project-host"
version = "0.1.0"
edition = "2021"

[dependencies]
zisk-prover = "0.1"
serde = { version = "1.0", features = ["derive"] }
```

### CLI-Based Proving (Recommended)
```bash
# Build the guest program
cargo zisk build --manifest-path guest/Cargo.toml

# Run without proof (testing)
cargo zisk run --manifest-path guest/Cargo.toml --input input.bin

# Generate proof
cargo zisk prove --manifest-path guest/Cargo.toml --input input.bin

# Verify proof
cargo zisk verify --proof proof.bin
```

### src/main.rs — SDK-Based Proving
```rust
use zisk_prover::{Prover, Input};

fn main() {
    // Load compiled guest ELF
    let elf = std::fs::read("./guest/target/riscv/release/guest").unwrap();

    // Prepare inputs
    let mut input = Input::new();
    input.write(&my_input);          // Serde-serialized
    // input.write_bytes(&raw_data); // Raw bytes

    // Generate proof
    let prover = Prover::new();
    let proof = prover.prove(&elf, &input).unwrap();

    // Verify
    prover.verify(&proof).unwrap();

    // Extract public output
    let output_bytes = proof.public_output();
    println!("Proof verified, output: {} bytes", output_bytes.len());
}
```

**TRANSFER NOTE**: Zisk is newer and its SDK API may change. The CLI-based approach is recommended for initial transfers. The guest must be `#![no_std]`.

---

## Jolt Host Template

### Cargo.toml
```toml
[package]
name = "my-project-host"
version = "0.1.0"
edition = "2021"

[dependencies]
jolt-sdk = "0.1"
my-guest = { path = "../guest" }  # The guest library crate
```

### src/main.rs
```rust
use my_guest::compute;  // The #[jolt::provable] function

fn main() {
    // Jolt proves function calls directly — no ELF loading
    let (prove, verify) = my_guest::build_compute();

    // Provide inputs as function arguments
    let input = MyInput { /* ... */ };
    let (output, proof) = prove(input);

    // Verify
    let is_valid = verify(proof);
    assert!(is_valid, "Proof verification failed");

    println!("Verified output: {:?}", output);
}
```

**TRANSFER NOTE**: Jolt's model is fundamentally different — it proves function executions, not program runs. The guest is a library crate with `#[jolt::provable]` functions. There is no ELF, no stdin, no streaming I/O. All inputs must be function parameters, all outputs must be return values. See `references/jolt-special.md` for restructuring guidance.

---

## Host-Side Input Ordering

**Critical**: The order of `write()` / `write_slice()` calls in the host must exactly match the order of `read()` / `read_slice()` / `read_vec()` calls in the guest. This is true for ALL platforms.

When transferring, preserve the exact input submission order:

```rust
// If guest reads in this order:
let config: Config = read();        // 1st read
let weights: Vec<u8> = read_vec();  // 2nd read
let prompt: String = read();        // 3rd read

// Then host MUST write in this order:
stdin.write(&config);               // 1st write
stdin.write_slice(&weights);        // 2nd write
stdin.write(&prompt);               // 3rd write
```

---

## Freivalds Host Hint Pattern

For Freivalds-optimized guests (like verifiable inference), the host must:
1. Run a native forward pass to compute results
2. Send pre-truncation values as hints to the guest

### R0VM Freivalds Host
```rust
let mut env_builder = ExecutorEnv::builder();
// Submit weights + inputs normally
env_builder.write_slice(&weight_words);
env_builder.write(&input);
// Submit pre-computed i64 hints for Freivalds verification
env_builder.write_slice(&hint_i64_as_u32_words);
let env = env_builder.build().unwrap();
```

### SP1 Freivalds Host
```rust
let mut stdin = SP1Stdin::new();
// Submit weights + inputs normally
stdin.write_slice(&weight_bytes);
stdin.write(&input);
// TRANSFER NOTE: SP1 has a dedicated hint channel for unproven data.
// Use write_slice for proven inputs, but consider hint writes for Freivalds data.
stdin.write_slice(&hint_i64_bytes);
```

**TRANSFER NOTE**: When transferring Freivalds-optimized code, the host's native forward pass logic (computing hints) is pure Rust computation that doesn't need translation — only the I/O submission calls change. However, the byte format of hints may need adjustment if the source uses u32-aligned writes and the target uses byte-aligned writes.
