# zkVM Guest Program Transfer Skill

## Metadata
- **Name**: zkVM-Guest-Program-Transfer
- **Description**: Transfer guest programs between zkVM platforms (R0VM, SP1, OpenVM, Zisk, Jolt)
- **Keywords**: zkvm, transfer, migrate, port, risc0, sp1, openvm, zisk, jolt, guest, prover, proof

## When to Use

Activate this skill when the user mentions:
- Transferring, migrating, or porting a zkVM guest program to another platform
- Converting between R0VM / SP1 / OpenVM / Zisk / Jolt
- "zkvm transfer", "port to SP1", "migrate from risc0", "convert guest program"
- Wanting to run the same computation on a different proving system

## Invocation

```
/zkvm-transfer <source-platform> <target-platform> [--guest <path>] [--host <path>] [--output <dir>]
```

**Platforms**: `r0vm`, `sp1`, `openvm`, `zisk`, `jolt`

If paths are not provided, ask the user to point to their source guest and host code.

## Workflow

### Phase 1: Analysis

1. Read the source guest program (`main.rs` or `lib.rs`) and host program
2. Identify every zkVM-specific API call and classify each:
   - **Entry point**: `entry!()`, `entrypoint!()`, `#[openvm::main]`, `#[jolt::provable]`
   - **Structured I/O**: `env::read()`, `env::commit()`, and equivalents
   - **Raw I/O**: `env::read_slice()`, `env::commit_slice()`, raw byte reads
   - **Cycle counting**: `env::cycle_count()` and equivalents
   - **Hints/advice**: Host-to-guest hint channels
   - **Precompiles**: SHA-256, Keccak, BigInt, etc.
   - **Prover calls**: `prove()`, `verify()`, receipt handling
3. Catalog all pure computation code (no translation needed)
4. Note any `#![no_main]`, `#![no_std]`, allocator setup, feature flags

### Phase 2: Compatibility Assessment

Before generating code, present a summary to the user:

```
## Transfer Assessment: <source> → <target>

### Direct translations (N items)
- [list each API call and its target equivalent]

### Requires adaptation (N items)
- [list items needing non-trivial changes, with explanation]

### Not supported on target (N items)
- [list features that cannot translate, with workarounds if any]

### Warnings
- [memory model differences, I/O alignment, missing precompiles, etc.]
```

**Key compatibility concerns to check:**
- **Jolt target**: Cannot do streaming I/O, no `main()`-based programs — see `references/jolt-special.md`
- **Raw I/O alignment**: R0VM reads are u32-word-aligned; SP1/OpenVM return `Vec<u8>`
- **Precompile availability**: SHA-256 accelerated on R0VM/SP1, may not exist on Zisk/Jolt
- **Memory limits**: Vary by platform; large model weight streaming may not work everywhere
- **`no_std` requirements**: Zisk requires `#![no_std]`; others support `std`

Wait for user confirmation before proceeding to code generation.

### Phase 3: Code Generation

Generate the complete target project using templates from `references/`:

```
<output-dir>/
  guest/
    Cargo.toml          # From references/build-systems.md
    src/main.rs          # From references/guest-patterns.md (or lib.rs for Jolt)
    build.rs             # If required by target
  host/
    Cargo.toml          # From references/build-systems.md
    src/main.rs          # From references/host-patterns.md
    build.rs             # If required by target
  Cargo.toml            # Workspace root (if applicable)
  TRANSFER-NOTES.md     # All changes, warnings, manual steps
```

**Code generation rules:**
1. Preserve ALL pure computation code exactly — do not refactor or "improve" it
2. Only change zkVM-specific API calls using the mappings in `references/api-mapping.md`
3. Add `// TRANSFER NOTE: <explanation>` comments where behavior differs between platforms
4. For raw I/O helpers (`read_i8_raw`, etc.), rewrite using target's byte-reading API
5. Stub unsupported features with `// TRANSFER NOTE: Not available on <target>` and a reasonable fallback
6. Generate `TRANSFER-NOTES.md` documenting every change, warning, and manual step needed

### Phase 4: Verification

Provide the user with:
1. **Build commands** to compile the generated project
2. **Expected output** — what a successful build looks like
3. **Test strategy** — how to verify the transfer produces the same results
4. **Known issues** — anything that needs manual attention

```
## Next Steps

1. Install toolchain: <command>
2. Build guest: <command>
3. Build host: <command>
4. Run with test input: <command>
5. Manual steps required: [list from TRANSFER-NOTES.md]
```

## Reference Files

- `references/api-mapping.md` — Complete cross-platform API mapping tables
- `references/guest-patterns.md` — Guest program translation templates
- `references/host-patterns.md` — Host/prover SDK templates
- `references/build-systems.md` — Build configuration and toolchain reference
- `references/jolt-special.md` — Jolt's function-level proving model and restructuring guide
