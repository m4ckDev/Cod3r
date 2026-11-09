# CMake basics cheat sheet

Modern, target-based CMake (3.16+). Describe each library or executable as a target and attach its properties to the target. Avoid global `include_directories` / `add_definitions`.

## Project layout used below

```text
demo/
├── CMakeLists.txt
├── include/greeter/greeter.h
├── src/greeter.cpp
├── src/main.cpp
└── tests/test_greeter.cpp
```

## Minimal CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.16)
project(demo VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)             # -std=c++17, not gnu++17

# Default to a Release build for single-config generators (Makefiles, Ninja)
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_BUILD_TYPE Release)
endif()

option(DEMO_SANITIZE "Build with ASan + UBSan" OFF)

# A library target
add_library(greeter src/greeter.cpp)
target_include_directories(greeter PUBLIC include)   # PUBLIC: users of greeter also get this
target_compile_options(greeter PRIVATE -Wall -Wextra -Wpedantic)

# An executable that links the library
add_executable(demo src/main.cpp)
target_link_libraries(demo PRIVATE greeter)
target_compile_options(demo PRIVATE -Wall -Wextra -Wpedantic)

# Tests
enable_testing()
add_executable(test_greeter tests/test_greeter.cpp)
target_link_libraries(test_greeter PRIVATE greeter)
add_test(NAME greeter COMMAND test_greeter)

# Sanitizers: every target must get the flags, including tests
if(DEMO_SANITIZE)
  foreach(t greeter demo test_greeter)
    target_compile_options(${t} PRIVATE -fsanitize=address,undefined -fno-omit-frame-pointer)
    target_link_options(${t} PRIVATE -fsanitize=address,undefined)
  endforeach()
endif()
```

## Source files

`include/greeter/greeter.h`

```cpp
#pragma once
#include <string>

namespace greeter {
std::string greet(const std::string& name);
}
```

`src/greeter.cpp`

```cpp
#include "greeter/greeter.h"

namespace greeter {
std::string greet(const std::string& name) { return "Hello, " + name + "!"; }
}  // namespace greeter
```

`src/main.cpp`

```cpp
#include <iostream>

#include "greeter/greeter.h"

int main() {
  std::cout << greeter::greet("CMake") << '\n';
}
```

`tests/test_greeter.cpp`

```cpp
#include <cstdlib>
#include <iostream>

#include "greeter/greeter.h"

int main() {
  if (greeter::greet("Ada") != "Hello, Ada!") {
    std::cerr << "FAIL\n";
    return EXIT_FAILURE;       // non-zero exit = test failed
  }
  std::cout << "PASS\n";
}
```

Only one `main()` may exist per executable target.

## Configure, build, test

```bash
cmake -S . -B build                           # configure (source dir, build dir)
cmake --build build -j                        # compile
ctest --test-dir build --output-on-failure    # run tests
./build/demo

cmake -S . -B build-dbg -DCMAKE_BUILD_TYPE=Debug -DDEMO_SANITIZE=ON   # sanitizer build
cmake -S . -B build -G Ninja                  # use Ninja instead of Make
cmake --install build --prefix /tmp/demo-install   # needs install() rules
```

Always build out of source (`-B build`). Deleting `build/` fully resets state.

## Keywords: PUBLIC / PRIVATE / INTERFACE

| Keyword | Used to build the target? | Propagated to targets that link it? |
|---|---|---|
| `PRIVATE` | Yes | No |
| `PUBLIC` | Yes | Yes |
| `INTERFACE` | No | Yes (header-only libs) |

Rule: if the dependency appears in your public headers, use `PUBLIC`. If it appears only in `.cpp` files, use `PRIVATE`.

## Dependencies and other common commands

```cmake
find_package(Threads REQUIRED)
target_link_libraries(demo PRIVATE Threads::Threads)

# System library via pkg-config
find_package(PkgConfig REQUIRED)
pkg_check_modules(ZLIB REQUIRED IMPORTED_TARGET zlib)
target_link_libraries(demo PRIVATE PkgConfig::ZLIB)

# Many sources
target_sources(demo PRIVATE src/a.cpp src/b.cpp)

# Preprocessor definition and per-target feature requirement
target_compile_definitions(demo PRIVATE APP_VERSION="1.0")
target_compile_features(demo PUBLIC cxx_std_17)

# Header-only library
add_library(hdr INTERFACE)
target_include_directories(hdr INTERFACE include)

# Install rules
install(TARGETS demo RUNTIME DESTINATION bin)

# Debug a build
message(STATUS "Compiler: ${CMAKE_CXX_COMPILER_ID} ${CMAKE_CXX_COMPILER_VERSION}")
```

Useful cache variables: `-DCMAKE_BUILD_TYPE=Debug|Release|RelWithDebInfo`, `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON` (writes `compile_commands.json` for clangd), `-DCMAKE_CXX_COMPILER=clang++`.

## Common mistakes

| Mistake | Why it hurts | Fix |
|---|---|---|
| In-source build (`cmake .`) | Litters the source tree with generated files | `cmake -S . -B build` |
| `include_directories()` / `link_libraries()` globally | Leaks settings to every target | `target_include_directories` / `target_link_libraries` |
| `file(GLOB ...)` for sources | New files are not noticed until re-configure | List sources explicitly |
| Setting `CMAKE_CXX_FLAGS` to add `-std=c++17` | Breaks on other compilers | `CMAKE_CXX_STANDARD` or `target_compile_features` |
| Forgetting `CMAKE_CXX_EXTENSIONS OFF` | Gets `-std=gnu++17`, non-portable code compiles | Set it to `OFF` |
| Wrong `PUBLIC`/`PRIVATE` | Missing includes for users, or leaked internals | Public headers need `PUBLIC` |
| No `CMAKE_BUILD_TYPE` | Unoptimized, no flags (empty type) | Default it to `Release` or pass `-DCMAKE_BUILD_TYPE=` |
| Changing options but seeing no effect | Stale `CMakeCache.txt` | Re-run with `-D...` or delete `build/` |
| Two `main()` in one executable | Linker error: multiple definition | One `main` per executable target |
| Calling `enable_testing()` in a subdirectory only | `ctest` finds no tests from the top | Call it in the top-level `CMakeLists.txt` |
