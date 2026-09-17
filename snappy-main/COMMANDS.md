# Build, Test, and Benchmark Commands (Windows / PowerShell)

Verified on Windows 11 with CMake 4.3.2, Ninja 1.13.2, and MinGW g++ 16.1.0.
There is no MSVC or clang on this machine, so all commands below target the
MinGW toolchain via the Ninja generator.

Run every command from the repository root:

```powershell
cd C:\Users\aiDAPTIV\Documents\benchmarks\snappy-main\snappy-main
```

## Prerequisite: vendored dependencies

`third_party\googletest` and `third_party\benchmark` ship as empty submodule
directories, and this folder is not a git checkout, so the `git submodule
update --init` step from the README has nothing to read. Clone them directly:

```powershell
git clone --depth 1 https://github.com/google/googletest.git third_party\googletest
git clone --depth 1 https://github.com/google/benchmark.git third_party\benchmark
```

Without these, CMake configuration fails: `snappy_unittest` needs gmock/gtest
and `snappy_benchmark` needs benchmark_main.

## Configure

Run once per build directory. Re-running it is harmless.

```powershell
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_COMPILER=gcc -DCMAKE_CXX_COMPILER=g++
```

The `benchmark` subproject prints a warning about `std::regex` with exceptions
disabled. It is expected and does not affect the build.

## Compile

```powershell
cmake --build build
```

A clean build runs 44 steps and produces three executables in `build\`:
`snappy_unittest.exe`, `snappy_benchmark.exe`, and `snappy_test_tool.exe`.
Warnings about a redefined `NOMINMAX` (snappy-test.h) and a deprecated
`benchmark::internal::Benchmark` alias (snappy_benchmark.cc) are expected.

To rebuild from scratch, delete the build directory first:

```powershell
Remove-Item -Recurse -Force build
```

## Test

```powershell
ctest --test-dir build --output-on-failure
```

Expected: `100% tests passed, 0 tests failed out of 1` (the `snappy_unittest`
target). Exits 0 on success.

To run the test binary directly instead of through ctest:

```powershell
.\build\snappy_unittest.exe
```

## Benchmark

Full suite:

```powershell
.\build\snappy_benchmark.exe
```

A quick subset while iterating, plus how to discover benchmark names:

```powershell
.\build\snappy_benchmark.exe --benchmark_min_time=0.05s --benchmark_filter="BM_UFlat/[012]/1"
.\build\snappy_benchmark.exe --benchmark_list_tests
```

Reference numbers from one run (16-core, 3802 MHz):

| Benchmark      | Corpus | Throughput   |
| -------------- | ------ | ------------ |
| `BM_UFlat/0/1` | html   | 10.2 GiB/s   |
| `BM_UFlat/1/1` | urls   | 2.80 GiB/s   |
| `BM_UFlat/2/1` | jpg    | 52.6 GiB/s   |

## Comparative test tool

Benchmarks Snappy against other compression libraries detected at configure
time. zlib was detected on this machine; LZO and LZ4 were not, and those flags
report "library not compiled in".

```powershell
.\build\snappy_test_tool.exe testdata\html
.\build\snappy_test_tool.exe --zlib testdata\html
```

## Gotchas

- **Working directory matters.** `snappy_benchmark.exe` and
  `snappy_test_tool.exe` resolve `testdata/` relative to the current directory.
  Launch them from the repository root, not from inside `build\`, or they abort
  with `testdata/html: No such file or directory`. `ctest` is unaffected because
  CMakeLists.txt pins the test's working directory to the source tree.
- **Benchmark names carry two indices** (`BM_UFlat/0/1`), so a filter such as
  `BM_UFlat/[0-3]$` silently matches nothing. Use `--benchmark_list_tests` to
  see the real names.
- **Google Benchmark writes its banner to stderr.** Under PowerShell 5.1,
  redirecting with `2>&1` wraps those lines in `NativeCommandError` records even
  on a successful run. Do not redirect, and the output reads normally.
