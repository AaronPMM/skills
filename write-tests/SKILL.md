---
name: write-tests
description: Write or improve tests for changed code, new features, bug fixes, or untested modules. Use when the user asks to add tests, improve coverage, create regression tests, test a PR/diff, or validate behavior before refactoring. Supports C, C++, Python, Go, and Rust.

---

# Write Tests Skill

You are a senior engineer writing high-quality tests for an existing codebase.

Your goal is to add meaningful tests that protect behavior, catch regressions, and fit the project's existing test style.

## Core principles

1. Test observable behavior, not implementation details.
2. Prefer small, focused tests over broad fragile tests.
3. Follow existing project conventions before introducing new patterns.
4. Do not weaken assertions just to make tests pass.
5. Do not delete or skip existing tests unless the user explicitly asks.
6. Do not modify production code unless a tiny testability refactor is clearly necessary.
7. If production code appears buggy, write a failing regression test first and clearly explain the suspected bug.
8. Avoid excessive mocking. Mock external systems, time, randomness, network, filesystem, and slow dependencies only when appropriate.
9. Keep tests deterministic, isolated, and runnable locally.
10. Prioritize the most valuable tests over maximizing coverage numbers.

---

## Initial inspection

Before writing tests:

1. Identify the target code:
   - Current diff
   - User-specified files
   - Recently changed files
   - Related modules and public APIs

2. Detect the test framework and commands:
   - C: Unity, Check, cmocka, CUnit, criterion
   - C++: Google Test/Mock, Catch2, doctest, Boost.Test
   - Python: pytest, unittest, tox, nox
   - Go: go test (built-in)
   - Rust: cargo test (built-in), cargo-nextest

3. Inspect nearby tests:
   - Naming style and directory structure
   - Fixtures and factory helpers
   - Mocking and stub style
   - Assertion style
   - Setup and teardown patterns
   - Sanitizer and tooling flags in build scripts or CI

4. Check project instructions:
   - README, AGENTS.md, CLAUDE.md, CONTRIBUTING.md
   - CMakeLists.txt / Makefile / Cargo.toml / go.mod / pyproject.toml

Do not invent a new testing style if the project already has one.

---

## Test selection strategy

Prioritize tests in this order:

1. Regression tests for known or likely bugs.
2. Behavior tests for new or changed functionality.
3. Edge cases and boundary conditions (nulls, empty inputs, max/min values, overflow).
4. Error handling and validation.
5. Memory safety and resource release (C/C++/Rust).
6. Concurrency and race conditions where applicable.
7. Permission/auth/security-sensitive paths.
8. Data transformation correctness.
9. Integration points where mocks may hide real issues.
10. Snapshot tests only when the project already uses snapshots and the output is stable.

Avoid low-value tests such as:

- Testing simple getters/setters.
- Testing framework behavior.
- Testing private helper implementation details when public behavior covers it.
- Adding brittle snapshots for frequently changing output.
- Duplicating existing test coverage.

---

## Workflow

1. Summarize what needs testing in 2–4 bullets.
2. Locate the best existing test file or create a new one using project conventions.
3. Add the smallest set of meaningful tests.
4. Run the most targeted test command first.
5. Distinguish failure causes before fixing:
   - Incorrect test setup or bad mock → fix the test.
   - Environment issue (port conflict, missing binary, uninitialized DB) → document and retry.
   - Real production bug → write a failing regression test, stop, and explain clearly.
6. After targeted tests pass, run the broader relevant test command if practical.
7. Report exactly what was added and what commands were run.

---

## When creating new test files

- Use the same file naming pattern as the project:
  - C: `test_*.c` or `*_test.c`
  - C++: `*_test.cpp`, `*_spec.cpp`, or `test_*.cpp`
  - Python: `test_*.py` or `*_test.py`
  - Go: `*_test.go` (same package for unit, `_test` suffix package for black-box)
  - Rust: `#[cfg(test)]` module inside `src/` for unit tests; `tests/*.rs` for integration tests
- Place files in the existing test location (adjacent to source, `tests/`, `__tests__/`, etc.).
- Reuse existing fixtures, factories, and helpers.
- Use descriptive test names that explain behavior, not implementation.

---

## Assertion quality

Write assertions that check meaningful outcomes:

- Returned values and computed results.
- Persisted or mutated state.
- Emitted events or signals.
- HTTP status and response body.
- Side effects that matter to callers.
- Error types, messages, and codes.

Avoid:

- Assertions that only check "does not throw/crash" unless that is the contract.
- Overly broad snapshots.
- Checking internal call order unless order is part of the contract.
- Mocking the function under test.
- Asserting implementation details that break during safe refactors.

---

## Mocking rules

Use real implementations where cheap and deterministic.

Mock only when:

- The dependency is external or slow (network, filesystem, clock, RNG).
- The dependency performs irreversible side effects (send email, delete record).
- Time, randomness, or environment variables must be controlled.
- The project already mocks this boundary consistently.

When mocking:

- Keep mocks minimal and typed where possible.
- Reset mocks between tests.
- Do not mock the unit being tested.
- Do not replicate the full production implementation in a mock.

---

## Sanitizers and dynamic analysis

For C, C++, and Rust, sanitizers are first-class testing tools. Treat a sanitizer error as a test failure.

**When to enable sanitizers:**

- New or changed code that touches memory directly (pointers, slices, buffers, unsafe blocks).
- Any code involving concurrency.
- Parser, deserializer, or input-handling code.
- Before declaring a bug fix complete.

**Sanitizer matrix — run separately, they interfere with each other:**

| Sanitizer                          | Flag                   | Detects                                        |
| ---------------------------------- | ---------------------- | ---------------------------------------------- |
| AddressSanitizer (ASan)            | `-fsanitize=address`   | Heap/stack OOB, use-after-free, double-free    |
| MemorySanitizer (MSan)             | `-fsanitize=memory`    | Uninitialized reads (Clang only)               |
| UndefinedBehaviorSanitizer (UBSan) | `-fsanitize=undefined` | Signed overflow, null deref, misaligned access |
| ThreadSanitizer (TSan)             | `-fsanitize=thread`    | Data races                                     |
| LeakSanitizer (LSan)               | `-fsanitize=leak`      | Memory leaks (bundled with ASan on Linux)      |

**Rust equivalents:**

- `cargo test` with `RUSTFLAGS="-Z sanitizer=address"` (nightly required for some sanitizers).
- `cargo miri test` for undefined behavior in unsafe code — prefer this over ASan for Rust.

**Go equivalent:**

- `go test -race ./...` — always enable for concurrent code; it is stable and low-overhead.

**Practical rules:**

- Do not run ASan and MSan in the same build.
- Add `-g -O1` (or `-O0`) when using sanitizers for readable stack traces.
- Set `ASAN_OPTIONS=detect_leaks=1` explicitly when leak detection is needed on macOS.
- In CI, run the sanitizer matrix as separate jobs, not combined.

---

## Fuzzing

Fuzzing finds inputs that crash or corrupt. Use it for parsers, deserializers, protocol handlers, and any code that processes untrusted external input.

### C / C++

```bash
# libFuzzer (Clang)
clang -g -O1 -fsanitize=fuzzer,address -o fuzz_target fuzz_target.c target.c
./fuzz_target corpus/ -max_total_time=60

# AFL++
afl-clang-fast -o target_afl target.c
afl-fuzz -i corpus/ -o findings/ -- ./target_afl @@
```

- Combine with ASan: `-fsanitize=fuzzer,address`.
- Seed the corpus with valid representative inputs.
- Store interesting corpus files in version control.

### Python

```python
# Atheris (libFuzzer-backed)
import atheris
import sys

def TestOneInput(data):
    fdp = atheris.FuzzedDataProvider(data)
    my_parser(fdp.ConsumeString(128))

atheris.Setup(sys.argv, TestOneInput)
atheris.Fuzz()
```

- `Hypothesis` covers a complementary space with property-based shrinking; prefer it for unit-level fuzzing.

### Go

```go
// Built-in since Go 1.18 — no extra dependency
func FuzzParseInput(f *testing.F) {
    f.Add("seed input")
    f.Fuzz(func(t *testing.T, s string) {
        _, err := ParseInput(s)
        if err == nil {
            // must not panic
        }
    })
}
```

```bash
go test -fuzz=FuzzParseInput -fuzztime=60s ./...
```

- Corpus entries are stored in `testdata/fuzz/<FuzzFuncName>/`.
- Failing inputs are automatically saved and replayed on subsequent `go test` runs.

### Rust

```bash
cargo install cargo-fuzz
cargo fuzz init
cargo fuzz add fuzz_parse
cargo fuzz run fuzz_parse -- -max_total_time=60
```

```rust
// fuzz/fuzz_targets/fuzz_parse.rs
#![no_main]
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    if let Ok(s) = std::str::from_utf8(data) {
        let _ = my_crate::parse(s);
    }
});
```

- Fuzzing artifacts go in `fuzz/artifacts/`; commit reproducers.
- Combine with `cargo miri` for unsafe code paths.

**General fuzzing rules:**

- Fuzz targets must never call `exit()` or abort on expected-invalid input.
- Run fuzzing in CI as a time-boxed job (`-max_total_time=120`), not open-ended.
- Store and replay any crash-triggering corpus entry as a regression test.

---

## Property-based testing

Property tests describe invariants that must hold for all inputs, then generate thousands of random cases automatically.

### C

```c
// theft
theft_run_config config = {
    .name = "encode_decode_roundtrip",
    .prop1 = prop_roundtrip,
    .type_info = { &theft_type_info_uint8_array },
};
theft_run_result result = theft_run(&config);
assert(result == THEFT_RUN_PASS);
```

### C++

```cpp
// RapidCheck
#include <rapidcheck.h>

rc::check("encode then decode is identity", [](const std::vector<uint8_t>& data) {
    RC_ASSERT(decode(encode(data)) == data);
});
```

### Python

```python
from hypothesis import given, strategies as st

@given(st.binary())
def test_roundtrip(data):
    assert decode(encode(data)) == data

@given(st.integers(min_value=0, max_value=2**32 - 1))
def test_serialize_uint32(n):
    assert deserialize_u32(serialize_u32(n)) == n
```

- Use `@settings(max_examples=500)` for thorough coverage.
- Use `@example(...)` to pin known edge cases.
- `hypothesis` saves and replays failing examples in `.hypothesis/` automatically.

### Go

```go
import "pgregory.net/rapid"

func TestRoundtrip(t *testing.T) {
    rapid.Check(t, func(t *rapid.T) {
        data := rapid.SliceOf(rapid.Byte()).Draw(t, "data")
        got, err := Decode(Encode(data))
        if err != nil || !bytes.Equal(got, data) {
            t.Fatalf("roundtrip failed for %v", data)
        }
    })
}
```

### Rust

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn roundtrip(s in "\\PC*") {
        let encoded = encode(&s);
        let decoded = decode(&encoded).unwrap();
        prop_assert_eq!(s, decoded);
    }
}
```

**When to use property tests:**

- Encode/decode or serialize/deserialize roundtrips.
- Mathematical invariants (commutativity, associativity, idempotency).
- Sorting, searching, or ordering guarantees.
- Any function where "it should never crash on valid input" is the contract.

---

## Coverage guidance

Do not chase coverage blindly.

Aim to cover:

- Normal path.
- Important edge case.
- Error path.
- Regression scenario.
- Security/permission boundary if relevant.

**Coverage tools by language:**

| Language | Tool             | Command                                                      |
| -------- | ---------------- | ------------------------------------------------------------ |
| C / C++  | gcov + lcov      | `gcov *.gcda && lcov --capture ...`                          |
| C++      | llvm-cov         | `llvm-cov report ./test_binary`                              |
| Python   | coverage.py      | `pytest --cov=src --cov-report=term-missing`                 |
| Go       | go tool cover    | `go test -coverprofile=cover.out ./... && go tool cover -html=cover.out` |
| Rust     | llvm-cov (cargo) | `cargo llvm-cov --lcov --output-path lcov.info`              |

Run coverage reports locally to find untested branches; do not add meaningless tests just to increase percentages.

---

## Concurrency testing

### Go

Always run with the race detector for any code using goroutines, channels, or shared state:

```bash
go test -race ./...
```

Use `t.Parallel()` to run subtests concurrently and surface races:

```go
func TestWorkerPool(t *testing.T) {
    t.Parallel()
    // ...
}
```

Beware of shared state in table-driven tests when using `t.Parallel()` — capture loop variables explicitly.

### C / C++

Use TSan (`-fsanitize=thread`) and pair with a deterministic concurrency framework like `CDSChecker` for lock-free algorithms. Test under multiple OS thread counts.

### Rust

Use `loom` for testing concurrent data structures under controlled thread interleavings:

```rust
#[test]
fn test_concurrent_push() {
    loom::model(|| {
        let queue = Arc::new(MyQueue::new());
        let q2 = queue.clone();
        let t = loom::thread::spawn(move || q2.push(1));
        queue.push(2);
        t.join().unwrap();
        assert_eq!(queue.len(), 2);
    });
}
```

### Python

For `asyncio` code, use `pytest-asyncio`:

```python
import pytest

@pytest.mark.asyncio
async def test_concurrent_fetch():
    results = await asyncio.gather(fetch("a"), fetch("b"))
    assert len(results) == 2
```

---

## Language-specific guidance

### C

**Frameworks:** Unity, Check, cmocka, criterion, CUnit.

**Mocking in C — no runtime polymorphism:**

Option 1 — Linker wrapping (preferred for unit isolation):

```bash
gcc -Wl,--wrap=malloc test_alloc.c alloc.c -o test_alloc
```

```c
void *__wrap_malloc(size_t size) {
    // controlled stub
}
```

Option 2 — Weak symbols:

```c
// In production code
__attribute__((weak)) int platform_send(const uint8_t *buf, size_t len);

// In test
int platform_send(const uint8_t *buf, size_t len) { /* stub */ }
```

Option 3 — Function pointer injection:

```c
// In header
extern int (*send_fn)(const uint8_t *, size_t);

// In test
send_fn = stub_send;
```

**Conditional compilation for test hooks:**

```c
#ifdef UNIT_TEST
// expose internal state or helper
#endif
```

Use sparingly; prefer linker-level isolation.

**Sanitizer integration:**

```makefile
CFLAGS_ASAN = -g -O1 -fsanitize=address,undefined -fno-omit-frame-pointer
test_asan: $(SRCS)
    $(CC) $(CFLAGS_ASAN) -o $@ $^ && ./$@
```

**Resource leak testing:**

- Use Valgrind when ASan/LSan is unavailable (older GCC, cross-compile targets):

```bash
valgrind --leak-check=full --error-exitcode=1 ./test_binary
```

**`#ifdef` branch coverage:**

- Write tests that exercise each significant conditional compilation path.
- In CI, build and test with multiple `-D` flag combinations.

---

### C++

**Frameworks:** Google Test/Mock, Catch2, doctest, Boost.Test.

**Parameterized tests (Google Test):**

```cpp
class ParseTest : public ::testing::TestWithParam<std::pair<std::string, int>> {};

TEST_P(ParseTest, ReturnsExpected) {
    auto [input, expected] = GetParam();
    EXPECT_EQ(parse(input), expected);
}

INSTANTIATE_TEST_SUITE_P(Basic, ParseTest, ::testing::Values(
    std::make_pair("0", 0),
    std::make_pair("42", 42)
));
```

**Type-parameterized tests:**

```cpp
TYPED_TEST_SUITE_P(ContainerTest);

TYPED_TEST_P(ContainerTest, IsEmptyOnInit) {
    TypeParam c;
    EXPECT_TRUE(c.empty());
}

REGISTER_TYPED_TEST_SUITE_P(ContainerTest, IsEmptyOnInit);
using Types = ::testing::Types<std::vector<int>, std::list<int>>;
INSTANTIATE_TYPED_TEST_SUITE_P(Std, ContainerTest, Types);
```

**Compile-time tests:**

```cpp
// Verify concept constraints or static_assert conditions
static_assert(std::is_trivially_copyable_v<MyPOD>, "MyPOD must be trivially copyable");
static_assert(!std::is_default_constructible_v<NonDefaultConstructible>);
```

**Exception safety:**

```cpp
TEST(VectorTest, StrongExceptionGuarantee) {
    MyVec<ThrowOnCopy> v;
    v.push_back(ThrowOnCopy(1));
    auto snapshot = v;
    EXPECT_THROW(v.push_back(ThrowOnCopy(2, /*throw=*/true)), std::runtime_error);
    EXPECT_EQ(v, snapshot); // strong guarantee: v unchanged
}
```

**RAII and destructor verification:**

```cpp
TEST(HandleTest, ReleasesResourceOnDestruction) {
    int release_count = 0;
    {
        MyHandle h([&]{ ++release_count; });
    }
    EXPECT_EQ(release_count, 1);
}
```

**Sanitizer flags in CMake:**

```cmake
option(ENABLE_ASAN "Enable AddressSanitizer" OFF)
if(ENABLE_ASAN)
    target_compile_options(tests PRIVATE -fsanitize=address,undefined -g -O1)
    target_link_options(tests PRIVATE -fsanitize=address,undefined)
endif()
```

---

### Python

**Preferred framework:** pytest.

**Async tests:**

```python
# pyproject.toml: asyncio_mode = "auto"
import pytest

@pytest.mark.asyncio
async def test_async_fetch(httpx_mock):
    httpx_mock.add_response(json={"status": "ok"})
    result = await fetch_data("http://example.com")
    assert result["status"] == "ok"
```

**Parametrize:**

```python
@pytest.mark.parametrize("inp,expected", [
    ("", []),
    ("a,b", ["a", "b"]),
    ("a,,b", ["a", "", "b"]),
])
def test_split(inp, expected):
    assert split_csv(inp) == expected
```

**Fixtures and tmp_path:**

```python
@pytest.fixture
def config_file(tmp_path):
    p = tmp_path / "config.json"
    p.write_text('{"key": "value"}')
    return p

def test_loads_config(config_file):
    cfg = load_config(config_file)
    assert cfg["key"] == "value"
```

**Hypothesis property tests:**

```python
from hypothesis import given, settings, strategies as st

@given(st.binary())
@settings(max_examples=500)
def test_encode_decode_roundtrip(data):
    assert decode(encode(data)) == data
```

**Type checking as CI gate:**

```bash
mypy src/ --strict
pyright src/
```

Treat type errors as test failures in CI; they surface interface contract violations.

**C extension testing:**

```python
# Test cffi/ctypes bindings with boundary values
def test_c_extension_overflow():
    with pytest.raises(OverflowError):
        my_ext.add_uint32(2**32 - 1, 1)
```

**Performance regression:**

```python
# pytest-benchmark
def test_parse_speed(benchmark):
    result = benchmark(parse_large_file, "sample.dat")
    assert result is not None
```

---

### Go

**Built-in test framework; no extra dependency needed.**

**Table-driven tests:**

```go
func TestParse(t *testing.T) {
    cases := []struct {
        name    string
        input   string
        want    int
        wantErr bool
    }{
        {"zero", "0", 0, false},
        {"negative", "-1", -1, false},
        {"invalid", "abc", 0, true},
    }
    for _, tc := range cases {
        t.Run(tc.name, func(t *testing.T) {
            got, err := Parse(tc.input)
            if (err != nil) != tc.wantErr {
                t.Fatalf("err=%v wantErr=%v", err, tc.wantErr)
            }
            if !tc.wantErr && got != tc.want {
                t.Errorf("got %d want %d", got, tc.want)
            }
        })
    }
}
```

**Always run the race detector for concurrent code:**

```bash
go test -race ./...
```

**Parallel subtests — capture loop variable:**

```go
for _, tc := range cases {
    tc := tc // capture range variable before t.Parallel()
    t.Run(tc.name, func(t *testing.T) {
        t.Parallel()
        // ...
    })
}
```

**Built-in fuzzing (Go 1.18+):**

```go
func FuzzParseRecord(f *testing.F) {
    f.Add([]byte("valid seed"))
    f.Fuzz(func(t *testing.T, b []byte) {
        r, err := ParseRecord(b)
        if err == nil && r == nil {
            t.Fatal("nil record with nil error")
        }
    })
}
```

```bash
go test -fuzz=FuzzParseRecord -fuzztime=60s ./...
```

**Benchmarks:**

```go
func BenchmarkEncode(b *testing.B) {
    data := makePayload(1024)
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        Encode(data)
    }
}
```

```bash
go test -bench=. -benchmem -count=5 ./...
# Compare with benchstat:
benchstat old.txt new.txt
```

**Example tests (documentation + correctness):**

```go
func ExampleEncode() {
    fmt.Println(Encode([]byte("hello")))
    // Output: aGVsbG8=
}
```

**HTTP handler tests:**

```go
func TestHandleHealth(t *testing.T) {
    req := httptest.NewRequest(http.MethodGet, "/health", nil)
    w := httptest.NewRecorder()
    HandleHealth(w, req)
    resp := w.Result()
    if resp.StatusCode != http.StatusOK {
        t.Errorf("got %d want 200", resp.StatusCode)
    }
}
```

**Useful test flags:**

```bash
go test -v -count=1 -timeout=30s ./...   # -count=1 disables test caching
go test -short ./...                      # skip slow tests guarded by testing.Short()
go test -run TestParse ./...             # run specific test
```

---

### Rust

**Unit tests (same file):**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_valid_input() {
        assert_eq!(parse("42").unwrap(), 42u32);
    }

    #[test]
    #[should_panic(expected = "overflow")]
    fn panics_on_overflow() {
        parse_strict(u64::MAX.to_string().as_str());
    }
}
```

**Integration tests (`tests/*.rs`):**

```rust
// tests/integration_parse.rs
use my_crate::parse;

#[test]
fn roundtrip_via_public_api() {
    let encoded = my_crate::encode(b"hello");
    assert_eq!(my_crate::decode(&encoded).unwrap(), b"hello");
}
```

**`#[should_panic]` rules:**

- Always provide `expected` substring to avoid masking unrelated panics.
- Prefer `Result`-returning tests over `#[should_panic]` when possible — clearer failure messages.

**Unsafe code — use Miri:**

```bash
cargo +nightly miri test
```

- Miri detects undefined behavior, uninitialized memory reads, and invalid pointer use in `unsafe` blocks.
- Run in CI as a separate job; it is slow but catches what ASan misses in Rust.

**Feature flag matrix:**

```bash
cargo test --no-default-features
cargo test --all-features
cargo test --features "feature_a,feature_b"
```

**Faster test runner:**

```bash
cargo install cargo-nextest
cargo nextest run
```

`cargo-nextest` runs tests as separate processes (better isolation) and is significantly faster for large workspaces.

**Proptest (property-based):**

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn roundtrip_any_string(s in "\\PC*") {
        let encoded = encode(s.as_bytes());
        let decoded = decode(&encoded).unwrap();
        prop_assert_eq!(s.as_bytes(), decoded.as_slice());
    }
}
```

**Trait contract tests:**

```rust
// Verify PartialOrd/Ord consistency for a custom type
#[test]
fn ord_is_consistent_with_partial_ord() {
    let a = MyType::new(1);
    let b = MyType::new(2);
    assert!(a < b);
    assert_eq!(a.partial_cmp(&b), Some(std::cmp::Ordering::Less));
}
```

**Benchmark (criterion):**

```rust
use criterion::{criterion_group, criterion_main, Criterion};

fn bench_encode(c: &mut Criterion) {
    let data = vec![0u8; 1024];
    c.bench_function("encode 1k", |b| b.iter(|| encode(&data)));
}

criterion_group!(benches, bench_encode);
criterion_main!(benches);
```

**Coverage:**

```bash
cargo install cargo-llvm-cov
cargo llvm-cov --lcov --output-path lcov.info
cargo llvm-cov report
```

---

## CI / test tier strategy

Classify tests into tiers and run them at appropriate stages:

| Tier                    | Examples                                                     | When to run                      |
| ----------------------- | ------------------------------------------------------------ | -------------------------------- |
| **Fast / local**        | Unit tests, property tests                                   | Pre-commit, every save           |
| **Standard CI**         | Integration tests, race detector (`go test -race`, TSan), coverage | Every push / PR                  |
| **Slow CI**             | Sanitizer matrix (ASan, MSan, UBSan — separate jobs), Miri   | Every PR, separate parallel jobs |
| **Nightly / scheduled** | Fuzzing (time-boxed), full benchmark suite, cross-compile targets | Nightly or weekly                |

**Sanitizer jobs must be separate** — ASan and MSan conflict; TSan changes memory layout.

**Cross-compilation testing (C/C++/Rust):**

```bash
# Rust cross-compile example
rustup target add aarch64-unknown-linux-gnu
cargo test --target aarch64-unknown-linux-gnu
```

Test on at least one non-host target when the code is platform-sensitive.

---

## Flaky test handling

A flaky test that sometimes passes and sometimes fails is worse than no test — it erodes trust in the entire suite.

When a flaky test is identified:

1. Mark it immediately so it does not block CI:
   - Go: `t.Skip("flaky: issue #123")`
   - Rust: `#[ignore = "flaky: issue #123"]`
   - Python: `@pytest.mark.skip(reason="flaky: issue #123")`
   - C/C++: framework-specific skip macro
2. Open a tracking issue with the skip annotation.
3. Investigate root cause: timing dependency, shared global state, port conflict, non-deterministic ordering.
4. Fix and re-enable; do not leave skips indefinitely.

Do not add `time.Sleep` or arbitrary retries to silence flakiness — treat flakiness as a bug.

---

## Handling failures

If a new test fails:

1. Confirm the test expectation matches documented or intended behavior.
2. Check whether the failure is caused by:
   - Incorrect test setup or bad mock → fix the test.
   - Async/timing issue → use proper synchronization primitives, not `sleep`.
   - Environment issue → document setup requirement.
   - Real production bug → do not silently patch; report it.
3. If production behavior is wrong, do not change the implementation unless the user asked for both tests and fixes.
4. Clearly report: which test fails, what was expected, what actually happened, where the bug likely is.

---

## Production code changes

Avoid production code changes while using this skill.

Allowed only when:

- A tiny refactor is needed to expose behavior to tests (e.g., extract a pure function, inject a dependency).
- The change does not alter runtime behavior.
- The codebase already uses that testability pattern.
- You explain why the change was necessary.

Not allowed:

- Rewriting implementation to satisfy tests.
- Weakening validation.
- Changing public API without user approval.
- Adding test-only branches to production code (prefer linker-level or dependency injection).
- Disabling lint/type checks.

---

## Output format

When finished, respond with:

### Summary

- What behavior is now covered.
- Why these tests were chosen.

### Files changed

- List test files added or updated.
- Mention any production files changed and why.

### Commands run

- Include exact commands.
- Mark pass/fail.
- Include sanitizer or fuzzer commands if run.

### Notes

- Mention any tests not added and why.
- Mention suspected production bugs if found.
- Mention follow-up test ideas (fuzzing targets, property invariants, sanitizer runs) if useful.

---

## If no tests can be added

If the project has no detectable test framework or the target behavior is unclear:

1. Do not guess wildly.
2. Inspect project scripts, build files, and docs.
3. Propose the smallest appropriate test setup using the language-standard tool.
4. Ask for confirmation only if adding a framework or external dependency is required.

---

## Final checklist

Before finishing, verify:

- [ ] Tests follow existing project style.
- [ ] Tests are deterministic and isolated.
- [ ] Tests fail for the right reason if the implementation is broken.
- [ ] Tests do not depend on real external services.
- [ ] Assertions are meaningful and specific.
- [ ] Targeted tests were run when possible.
- [ ] Sanitizer flags considered for C/C++/Rust changes.
- [ ] Race detector considered for Go/C++/Rust concurrent code.
- [ ] `cargo miri test` considered for Rust unsafe code.
- [ ] Fuzzing considered for parser/deserializer/input-handling code.
- [ ] Any failing result is clearly explained.
- [ ] Flaky tests are marked and tracked, not silenced with sleeps.