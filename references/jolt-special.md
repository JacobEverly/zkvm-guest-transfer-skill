# Jolt Special Considerations

## Jolt's Proving Model

Jolt proves **individual function executions**, not full programs. This is fundamentally different from R0VM/SP1/OpenVM/Zisk which prove entire program runs with streaming I/O.

### Key Differences
- **No `main()` entry point** — Jolt guests are library crates with `#[jolt::provable]` functions
- **No I/O API** — inputs are function parameters, outputs are return values
- **No streaming data** — all data must fit in function arguments/returns
- **No host interaction during execution** — no hints, no multi-round communication
- **Base RISC-V only** — `riscv32i` (no M extension), so multiply/divide is emulated

## When Jolt Transfer Works Well

Jolt transfers are straightforward when the source program follows this pattern:

```rust
// Source (R0VM):
fn main() {
    let input: MyInput = env::read();     // Read all input at once
    let result = pure_computation(input);  // Pure computation
    env::commit(&result);                  // Commit result
}
```

This maps directly to:

```rust
// Target (Jolt):
#[jolt::provable]
fn pure_computation(input: MyInput) -> MyResult {
    // Exact same computation code — no changes needed
    pure_computation(input)
}
```

### Good Candidates for Jolt Transfer
- Hash computations (SHA-256, Keccak)
- Signature verification
- Merkle proof verification
- Simple mathematical computations
- JSON/data parsing and validation
- Any single-pass input → output transformation

## When Jolt Transfer Is Difficult or Impossible

### Streaming I/O (Cannot Transfer)
Programs that read data incrementally cannot transfer to Jolt:

```rust
// This CANNOT transfer to Jolt:
fn main() {
    let n_chunks: u32 = env::read();
    for _ in 0..n_chunks {
        let chunk = env::read_slice(...);  // Multiple reads
        process(chunk);
    }
    env::commit(&result);
}
```

**Workaround**: Bundle all chunks into a single input struct:
```rust
#[jolt::provable]
fn process_all(all_chunks: Vec<Vec<u8>>) -> Result {
    // Process all chunks from the pre-bundled input
}
```

**Limitation**: This requires all data to fit in memory as function arguments, which may not be feasible for large datasets.

### Weight Streaming (Cannot Transfer)
ML inference programs that stream model weights layer-by-layer cannot transfer:

```rust
// This CANNOT transfer to Jolt:
fn main() {
    let config: Config = env::read();
    for layer in 0..num_layers {
        let weights = env::read_slice(...);  // Stream weights per layer
        output = forward_layer(output, weights);
    }
    env::commit(&output);
}
```

**No practical workaround**: Model weights (hundreds of MB to GB) cannot be passed as function arguments.

### Hint/Advice Channels (Cannot Transfer)
Programs using host hints for optimization (like Freivalds verification):

```rust
// This CANNOT transfer to Jolt:
fn main() {
    let input = env::read();
    let hint: Vec<i64> = env::read();  // Host-computed hint
    verify_with_freivalds(input, hint);
    env::commit(&result);
}
```

**Workaround**: Bundle hints into the function arguments:
```rust
#[jolt::provable]
fn verify(input: Input, hint: Vec<i64>) -> Result {
    verify_with_freivalds(input, hint)
}
```

This works if the hint data is small enough, but defeats the purpose if hints are large.

### Multi-Round Host Interaction (Cannot Transfer)
Programs with back-and-forth between host and guest:

```rust
// This CANNOT transfer to Jolt:
fn main() {
    let challenge = env::read();
    let response = compute_response(challenge);
    env::commit(&response);
    let next_challenge = env::read();  // Second read after commit
    // ...
}
```

**No workaround**: Jolt functions execute once with all inputs provided upfront.

## Restructuring Guide

### Step 1: Identify the Pure Computation Core
Extract the pure computation from the I/O wrapper:

```rust
// Before (R0VM):
fn main() {
    let a: u32 = env::read();
    let b: u32 = env::read();
    let result = a * b + 42;
    env::commit(&result);
}

// After identifying core:
// Pure computation: fn compute(a: u32, b: u32) -> u32 { a * b + 42 }
```

### Step 2: Define the Provable Function
```rust
// Jolt guest (lib.rs):
#[jolt::provable]
fn compute(a: u32, b: u32) -> u32 {
    a * b + 42
}
```

### Step 3: Bundle Multiple Reads Into Struct
If the source has multiple reads, create an input struct:

```rust
// Before (R0VM):
fn main() {
    let config: Config = env::read();
    let data: Vec<u8> = read_bytes_raw(n);
    let params: Params = env::read();
    let result = process(config, data, params);
    env::commit(&result);
}

// After (Jolt):
#[derive(Serialize, Deserialize)]
struct AllInputs {
    config: Config,
    data: Vec<u8>,
    params: Params,
}

#[jolt::provable]
fn process_all(inputs: AllInputs) -> Result {
    process(inputs.config, inputs.data, inputs.params)
}
```

### Step 4: Update the Host
```rust
// Jolt host:
use my_guest::process_all;

fn main() {
    let (prove, verify) = my_guest::build_process_all();

    let inputs = AllInputs {
        config: load_config(),
        data: load_data(),
        params: load_params(),
    };

    let (output, proof) = prove(inputs);
    assert!(verify(proof));
    println!("Result: {:?}", output);
}
```

## Performance Considerations

- **No M extension**: Jolt uses `riscv32i` (base integer only). Multiply and divide operations are emulated in software, making math-heavy programs significantly slower than on R0VM/SP1 (which use `riscv32im`).
- **Memory overhead**: All inputs must fit as function arguments, adding memory pressure compared to streaming approaches.
- **No precompile acceleration**: No accelerated SHA-256, Keccak, or BigInt operations. All crypto runs as standard RISC-V instructions.

## Transfer Assessment Template for Jolt

When assessing a transfer to Jolt, check each item:

- [ ] **Single read, single commit pattern?** → Direct transfer possible
- [ ] **Multiple reads?** → Can bundle into struct if total size is reasonable
- [ ] **Streaming/incremental I/O?** → Cannot transfer (or must load all data upfront)
- [ ] **Host hints/advice?** → Can bundle into input if size is small
- [ ] **Multi-round interaction?** → Cannot transfer
- [ ] **Heavy multiply/divide?** → Will be slower (no M extension)
- [ ] **Accelerated crypto?** → Will be slower (no precompiles)
- [ ] **Large memory footprint?** → May exceed practical limits as function args
