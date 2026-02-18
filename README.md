# zkVM Guest Program Transfer — Claude Code Skill

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that automates transferring guest programs between zkVM platforms: **R0VM (RISC Zero)**, **SP1 (Succinct)**, **OpenVM**, **Zisk**, and **Jolt**.

## What It Does

When you ask Claude Code to transfer a zkVM guest program from one platform to another, this skill provides:

- **Complete API mappings** between all 5 platforms (entry points, I/O, commits, hints, cycle counting)
- **Guest code translation** with correct entry point rewriting, I/O alignment, and precompile patches
- **Host/prover SDK templates** for each platform
- **Build system configs** (Cargo.toml, build.rs, toolchain setup)
- **`TRANSFER-NOTES.md`** documenting every change and known limitation

### Supported Platforms

| Platform | Guest Crate | Target Triple |
|----------|------------|---------------|
| R0VM (RISC Zero) | `risc0-zkvm ^3.0` | `riscv32im-risc0-zkvm-elf` |
| SP1 (Succinct) | `sp1-zkvm 4.0` | `riscv32im-succinct-zkvm-elf` |
| OpenVM | `openvm 1.0` | Custom RISC-V + extensions |
| Zisk | `ziskos 0.1` | Custom RISC-V |
| Jolt | `jolt-sdk 0.1` | `riscv32i-unknown-none-elf` |

## Install

### One-liner

```bash
mkdir -p ~/.claude/skills && ln -sf "$(pwd)" ~/.claude/skills/zkVM-Guest-Program-Transfer
```

### Step by step

1. Clone this repo:
   ```bash
   git clone https://github.com/JacobEverly/zkvm-guest-transfer-skill.git
   ```

2. Symlink into Claude Code's skill directory:
   ```bash
   mkdir -p ~/.claude/skills
   ln -sf /path/to/zkvm-guest-transfer-skill ~/.claude/skills/zkVM-Guest-Program-Transfer
   ```

3. Verify it's installed:
   ```bash
   ls ~/.claude/skills/zkVM-Guest-Program-Transfer/SKILL.md
   ```

That's it. Claude Code will automatically pick up the skill.

## Usage

In Claude Code, just describe what you want to transfer:

```
Transfer my SP1 guest program to R0VM
```

```
Port this RISC Zero guest to SP1: /path/to/my/guest/src/main.rs
```

```
Migrate my zkVM program from SP1 to OpenVM
```

The skill activates when you mention transferring, migrating, or porting between zkVM platforms.

### What Claude Does

1. **Analyzes** your source guest + host code, identifying all zkVM-specific API calls
2. **Assesses compatibility** and warns about non-translatable features before proceeding
3. **Generates** the complete target project (guest, host, Cargo.toml, build.rs, workspace)
4. **Documents** everything in `TRANSFER-NOTES.md` with build/run instructions

## File Structure

```
SKILL.md                          # Skill definition + 4-phase workflow
references/
  api-mapping.md                  # Cross-platform API reference tables
  guest-patterns.md               # Guest translation templates + real-world lessons
  host-patterns.md                # Host/prover SDK templates
  build-systems.md                # Build config, toolchains, Cargo.toml templates
  jolt-special.md                 # Jolt's function-level model + restructuring guide
```


## License

MIT
