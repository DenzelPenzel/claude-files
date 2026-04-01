---
description: "Generate and run fuzz tests for security vulnerabilities (Go, Rust)"
argument-hint: "<package_or_function_or_target> [--lang=go|rust] [--duration=10m] [--security-focus]"
allowed-tools:
  - Task
  - Bash
  - Read
  - Write
  - Edit
  - MultiEdit
  - Grep
  - Glob
---

# Fuzzing Campaign

Create and execute fuzz tests for: $ARGUMENTS

## Language Detection

Determine the project language before proceeding:
1. If `--lang=go` or `--lang=rust` is specified in the arguments, use that
2. If `Cargo.toml` exists in the project root or nearest ancestor, use **Rust**
3. If `go.mod` exists in the project root or nearest ancestor, use **Go**
4. If both exist, prefer whichever is closer to the target path; ask if ambiguous

Then follow the corresponding language section below.

---

## Go Fuzzing

### 1. Target Identification
   - Analyze functions that parse untrusted input
   - Identify security-critical code paths
   - Focus on: parsers, decoders, validators, network handlers

### 2. Fuzz Test Generation
   Generate comprehensive fuzz tests using Go's native fuzzing:
   ```go
   func FuzzTargetFunction(f *testing.F) {
       // Add seed corpus for better coverage
       f.Add(seedInput1)
       f.Add(edgeCaseInput)

       f.Fuzz(func(t *testing.T, input []byte) {
           // Security assertions
           defer func() {
               if r := recover(); r != nil {
                   t.Errorf("Panic detected: %v", r)
               }
           }()

           // Test target function
           result := TargetFunction(input)

           // Security invariants
           validateNoMemoryLeak(t)
           validateNoPanic(t)
           validateNoInfiniteLoop(t)
       })
   }
   ```

### 3. Security-Focused Test Patterns
   - **Input Validation**: Test with malformed/oversized inputs
   - **Resource Exhaustion**: Check for unbounded allocations
   - **State Corruption**: Verify state consistency after errors
   - **Injection Attacks**: Test special characters and escape sequences
   - **Integer Overflows**: Use boundary values and large numbers

### 4. Execution & Analysis
   ```bash
   # Run fuzzing with coverage
   go test -fuzz=FuzzTargetFunction -fuzztime=10m

   # Run all fuzz tests in package
   go test -fuzz=. -fuzztime=30m ./...

   # Check corpus coverage
   go test -cover -run=FuzzTargetFunction
   ```

### 5. Corpus Management
   - Save interesting inputs that increase coverage
   - Maintain regression corpus from crashes
   - Share corpus across team for continuous improvement

---

## Rust Fuzzing

### 1. Setup & Prerequisites
   - Verify `cargo-fuzz` is installed: `cargo install cargo-fuzz`
   - Initialize fuzz harness if absent: `cargo fuzz init`
   - Add a new fuzz target: `cargo fuzz add <target_name>`
   - Expected project structure:
     ```
     fuzz/
     ├── Cargo.toml            # Workspace member with libfuzzer-sys dependency
     ├── fuzz_targets/
     │   └── <target_name>.rs  # One file per fuzz target
     ├── corpus/<target_name>/ # Seed inputs per target
     └── artifacts/<target_name>/ # Crash reproductions
     ```
   - Ensure `fuzz/Cargo.toml` includes:
     ```toml
     [dependencies]
     libfuzzer-sys = "0.11"
     arbitrary = { version = "1", features = ["derive"] }
     ```

### 2. Fuzz Target Generation
   Generate fuzz targets using `libfuzzer-sys` and `arbitrary`:

   **Structured input** (preferred for functions with typed parameters):
   ```rust
   #![no_main]
   use libfuzzer_sys::fuzz_target;
   use arbitrary::Arbitrary;

   #[derive(Arbitrary, Debug)]
   struct FuzzInput {
       data: Vec<u8>,
       flags: u32,
   }

   fuzz_target!(|input: FuzzInput| {
       // Panics are automatically caught as crashes
       let _ = my_crate::process(&input.data, input.flags);
   });
   ```

   **Custom `Arbitrary` impl** for constrained domains:
   ```rust
   impl<'a> Arbitrary<'a> for BoundedInput {
       fn arbitrary(u: &mut arbitrary::Unstructured<'a>) -> arbitrary::Result<Self> {
           let len = u.int_in_range(1..=4096)?;
           let data = u.bytes(len)?.to_vec();
           Ok(BoundedInput { data })
       }
   }
   ```

   **Raw byte-slice target** for byte-oriented parsers:
   ```rust
   #![no_main]
   use libfuzzer_sys::fuzz_target;

   fuzz_target!(|data: &[u8]| {
       let _ = my_crate::parse_untrusted(data);
   });
   ```

### 3. Security-Focused Test Patterns
   - **Unsafe block testing**: Wrap calls to `unsafe` functions; detect UB via sanitizers
   - **Panic detection**: libFuzzer catches panics automatically; do not hide them with `catch_unwind`
   - **Differential fuzzing**: Compare implementations for identical output
     ```rust
     fuzz_target!(|data: &[u8]| {
         let safe = safe_impl(data);
         let fast = unsafe { optimized_impl(data) };
         assert_eq!(safe, fast);
     });
     ```
   - **Resource exhaustion**: Set `-rss_limit_mb=2048` and `-timeout=10` flags
   - **Integer overflows**: `cargo-fuzz` enables `-C overflow-checks=on` in release by default
   - **Input validation**: Test with malformed, oversized, and zero-length inputs

### 4. Execution & Analysis
   ```bash
   # Run a specific fuzz target (default duration until stopped)
   cargo fuzz run <target>

   # Run with a time limit
   cargo fuzz run <target> -- -max_total_time=600

   # Release mode for speed (overflow checks still on)
   cargo fuzz run <target> --release -- -max_total_time=1800

   # Run with AddressSanitizer for memory bugs (requires nightly)
   RUSTFLAGS="-Zsanitizer=address" cargo +nightly fuzz run <target>

   # Run with MemorySanitizer for uninitialized reads
   RUSTFLAGS="-Zsanitizer=memory" cargo +nightly fuzz run <target>

   # Generate coverage report from corpus
   cargo fuzz coverage <target>

   # Reproduce a specific crash
   cargo fuzz run <target> fuzz/artifacts/<target>/<crash_file>
   ```

### 5. Corpus Management
   ```bash
   # Minimize corpus (remove redundant inputs)
   cargo fuzz cmin <target>

   # Pretty-print a crash artifact
   cargo fuzz fmt <target> fuzz/artifacts/<target>/<crash_file>

   # Add seed inputs manually
   cp interesting_input fuzz/corpus/<target>/
   ```
   - Commit minimized corpus to the repository for CI reproducibility
   - Save crash artifacts for regression testing
   - Use `-max_len=<bytes>` to constrain input size when appropriate

---

Use the security-auditor agent to:
- Identify high-risk parsing functions
- Generate comprehensive fuzz harnesses
- Analyze crashes for security impact
- Create proof-of-concept exploits from crashes
- Recommend defensive coding patterns
