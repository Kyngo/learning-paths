---
title: "Build Systems & Tooling"
weight: 12
---

# Build Systems & Tooling

C and C++ have no standard build tool or package manager. The ecosystem relies on Make, CMake, and platform-specific tools. This section covers the build pipeline, debugging tools, sanitisers, and static analysis that make C/C++ development manageable.

---

## Make

The oldest and most universal build tool:

```makefile
CC = gcc
CXX = g++
CFLAGS = -Wall -Wextra -Werror -std=c17 -g
CXXFLAGS = -Wall -Wextra -Werror -std=c++17 -g
LDFLAGS =

SRCS = main.c user.c database.c
OBJS = $(SRCS:.c=.o)
TARGET = app

.PHONY: all clean test

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) $(LDFLAGS) -o $@ $^

%.o: %.c
	$(CC) $(CFLAGS) -c -o $@ $<

clean:
	rm -f $(OBJS) $(TARGET)

test: $(TARGET)
	./$(TARGET) --test
```

```bash
make          # build
make clean    # remove build artifacts
make -j$(nproc)  # parallel build
```

---

## CMake

The de facto standard for cross-platform C/C++ projects:

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(myapp VERSION 1.0 LANGUAGES C CXX)

set(CMAKE_C_STANDARD 17)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Compiler warnings
add_compile_options(-Wall -Wextra -Werror)

# Main executable
add_executable(myapp
    src/main.cpp
    src/user.cpp
    src/database.cpp
)

target_include_directories(myapp PRIVATE ${CMAKE_SOURCE_DIR}/include)

# External library
find_package(Threads REQUIRED)
target_link_libraries(myapp PRIVATE Threads::Threads)

# Tests
enable_testing()
add_executable(tests test/test_user.cpp)
target_link_libraries(tests PRIVATE Catch2::Catch2WithMain)
add_test(NAME unit_tests COMMAND tests)
```

```bash
mkdir build && cd build
cmake ..                    # configure
cmake --build .             # build
cmake --build . --target test  # run tests
cmake -DCMAKE_BUILD_TYPE=Release ..  # release build
```

### CMake Build Types

| Type | Flags | Use |
|------|-------|-----|
| `Debug` | `-g -O0` | Development, debugging |
| `Release` | `-O3 -DNDEBUG` | Production |
| `RelWithDebInfo` | `-O2 -g -DNDEBUG` | Production with debugging |
| `MinSizeRel` | `-Os -DNDEBUG` | Size-optimised |

---

## Package Managers

| Tool | Approach |
|------|----------|
| **vcpkg** (Microsoft) | System-wide or per-project. `vcpkg install fmt` |
| **Conan** | Python-based, Artifactory integration. `conan install .` |
| **CPM.cmake** | Header-only CMake module. Downloads deps at configure time |
| **FetchContent** | Built into CMake 3.11+. Downloads at configure time |

```cmake
# FetchContent example
include(FetchContent)
FetchContent_Declare(
    fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt
    GIT_TAG 10.1.1
)
FetchContent_MakeAvailable(fmt)
target_link_libraries(myapp PRIVATE fmt::fmt)
```

---

## Debugging with GDB

```bash
# Compile with debug symbols
gcc -g -O0 -o app main.c

# Start GDB
gdb ./app
```

### Essential GDB Commands

| Command | Shortcut | Action |
|---------|----------|--------|
| `run [args]` | `r` | Start program |
| `break main.c:42` | `b` | Set breakpoint |
| `break function_name` | `b` | Break at function entry |
| `next` | `n` | Step over (execute line) |
| `step` | `s` | Step into (enter function) |
| `continue` | `c` | Continue to next breakpoint |
| `print var` | `p` | Print variable value |
| `print *ptr` | `p` | Dereference and print |
| `backtrace` | `bt` | Show call stack |
| `frame N` | `f N` | Switch to stack frame N |
| `info locals` | | Show local variables |
| `watch var` | | Break when variable changes |
| `quit` | `q` | Exit GDB |

### LLDB (macOS)

Same concepts, slightly different commands:

| GDB | LLDB |
|-----|------|
| `run` | `run` |
| `break main.c:42` | `breakpoint set -f main.c -l 42` |
| `print x` | `print x` or `p x` |
| `backtrace` | `bt` |
| `next` | `next` or `n` |

---

## Sanitisers

Compiler-inserted runtime checks that catch bugs with minimal overhead:

### AddressSanitizer (ASan) — Memory Errors

```bash
gcc -fsanitize=address -g -o app main.c
./app  # crashes with detailed report on memory errors
```

Catches: buffer overflow, use-after-free, double-free, memory leaks.

### UndefinedBehaviorSanitizer (UBSan)

```bash
gcc -fsanitize=undefined -g -o app main.c
```

Catches: signed integer overflow, null pointer dereference, misaligned access, shift out of range.

### ThreadSanitizer (TSan) — Data Races

```bash
gcc -fsanitize=thread -g -o app main.c -pthread
```

Catches: data races, lock order violations.

### MemorySanitizer (MSan) — Uninitialised Reads

```bash
clang -fsanitize=memory -g -o app main.c  # Clang only
```

### Combined (Development Build)

```bash
gcc -g -O1 -fsanitize=address,undefined -fno-omit-frame-pointer main.c
```

**Always develop with sanitisers enabled.** They catch bugs that are invisible without them.

---

## Valgrind — Dynamic Analysis

```bash
valgrind --leak-check=full --show-leak-kinds=all ./app
```

| Tool | Checks |
|------|--------|
| Memcheck (default) | Memory leaks, invalid reads/writes, use of uninitialised values |
| Helgrind | Data races, lock ordering |
| Cachegrind | Cache hit/miss rates |
| Callgrind | Call graph profiling (use with KCachegrind for visualisation) |

Valgrind is slower than sanitisers (10-50× overhead) but catches different bugs and works without recompilation.

---

## Static Analysis

### Compiler Warnings (First Line of Defence)

```bash
gcc -Wall -Wextra -Wpedantic -Wshadow -Wformat=2 -Wconversion
```

### Clang Static Analyser

```bash
scan-build make         # wraps make with static analysis
scan-build cmake --build .
```

### Clang-Tidy (Linting)

```bash
clang-tidy main.cpp -- -std=c++17
clang-tidy -checks='*,-llvmlibc-*' main.cpp
```

Checks: modernisation (suggest C++17 features), performance, readability, bugprone patterns.

### Cppcheck

```bash
cppcheck --enable=all --std=c17 src/
```

---

## Profiling

```bash
# gprof
gcc -pg -o app main.c
./app
gprof app gmon.out > analysis.txt

# perf (Linux)
perf record ./app
perf report

# Instruments (macOS)
instruments -t "Time Profiler" ./app
```

---

## Key Takeaways

- CMake is the standard build system for non-trivial C/C++ projects. Makefiles are fine for small programs.
- `vcpkg` and `Conan` are the two main package managers. `FetchContent` is the CMake-native option.
- GDB (Linux) and LLDB (macOS) are essential. Learn `break`, `next`, `step`, `print`, `backtrace`.
- **Always develop with sanitisers** — ASan, UBSan, and TSan catch bugs that are invisible in normal execution.
- Valgrind catches memory leaks and uninitialised reads. Run it in CI if you can afford the slowdown.
- `clang-tidy` is the best C++ linter. Enable it in CI alongside compiler warnings at maximum level.
