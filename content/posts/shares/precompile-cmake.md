---
title: Precompile dependencies for cmake
summary: ""
date: 2024-12-04T21:10:00+08:00
table_of_contents: true
---

## Dependencies

C++ projects often requires dependencies especially when intending to build a fast demo. There are innumberable C++ libraries releasing as source code on github.

### Import a package

Unlike modern languages like go or rust, it's not an easy work to import a C++ package. C++ inherits the "old fashion style" build process from C, including several separated stages. To import a C++ package, you either:

- Import dependencies as source codes and build with your project at once.
- Build dependencies first and then link with the generated library file.

As a denpendency, the second way is more popular that a prebuild library can be shared by multiple projects, and any modification causing the project rebuilding would not affect dependency packages.

The project maintainer provides a target to the build tool named "install" to put the generated library files to a conventional install path, or specified by user. For C++ projects, the library files contains a boundle of header files and static or dynamic libraries. Importing a C++ package equals to including those headers and linking to libraries.

## CMake

Another noticeable fact is that [CMake](https://cmake.org/) is so popular that almost every C++ project offers a `CMakeLists.txt` file, which contains essential informations to build and install the package.

### Build source code

Below are the most common cmake commands to build and install, works as long as there's a `CMakeLists.txt` under the source path, use [GoogleTest](https://github.com/google/googletest) as example.
```
# confiture
cmake -DCMAKE_BUILD_TYPE=Release
      -DCMAKE_CXX_STANDARD=17
      -DCMAKE_INSTALL_PREFIX=<install path>
      -DCMAKE_INSTALL_LIBDIR=lib
      -S . -B build -G Ninja
# build
cmake --build build --config Release
# install
cmake --install build --prefix <install path>
```

> If compile on windows, don't forget to configure [msvc runtime library](https://cmake.org/cmake/help/latest/prop_tgt/MSVC_RUNTIME_LIBRARY.html). Use `-DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreaded` for `/MT` or `-DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreadedDLL` for `/MD`

After installing a package to the install path, there should be at least a header file and often library files. Whether cmake generates library files depends on whether the project is a header-only project.

This is the tree view of the install path.
```
# Under deps/out/Release of the demo project in the next section.
.
├── include
│   ├── gmock
│   │   ├── gmock-actions.h
│   │   ├── gmock-cardinalities.h
│   │   ├── gmock-function-mocker.h
│   │   ├── gmock-matchers.h
│   │   ├── gmock-more-actions.h
│   │   ├── gmock-more-matchers.h
│   │   ├── gmock-nice-strict.h
│   │   ├── gmock-spec-builders.h
│   │   ├── gmock.h
│   │   └── internal
│   │       ├── custom
│   │       │   ├── README.md
│   │       │   ├── gmock-generated-actions.h
│   │       │   ├── gmock-matchers.h
│   │       │   └── gmock-port.h
│   │       ├── gmock-internal-utils.h
│   │       ├── gmock-port.h
│   │       └── gmock-pp.h
│   └── gtest
│       ├── gtest-assertion-result.h
│       ├── gtest-death-test.h
│       ├── gtest-matchers.h
│       ├── gtest-message.h
│       ├── gtest-param-test.h
│       ├── gtest-printers.h
│       ├── gtest-spi.h
│       ├── gtest-test-part.h
│       ├── gtest-typed-test.h
│       ├── gtest.h
│       ├── gtest_pred_impl.h
│       ├── gtest_prod.h
│       └── internal
│           ├── custom
│           │   ├── README.md
│           │   ├── gtest-port.h
│           │   ├── gtest-printers.h
│           │   └── gtest.h
│           ├── gtest-death-test-internal.h
│           ├── gtest-filepath.h
│           ├── gtest-internal.h
│           ├── gtest-param-util.h
│           ├── gtest-port-arch.h
│           ├── gtest-port.h
│           ├── gtest-string.h
│           └── gtest-type-util.h
└── lib
    ├── cmake
    │   └── GTest
    │       ├── GTestConfig.cmake
    │       ├── GTestConfigVersion.cmake
    │       ├── GTestTargets-release.cmake
    │       └── GTestTargets.cmake
    ├── libgmock.a
    ├── libgmock_main.a
    ├── libgtest.a
    ├── libgtest_main.a
    └── pkgconfig
        ├── gmock.pc
        ├── gmock_main.pc
        ├── gtest.pc
        └── gtest_main.pc

12 directories, 52 files
```

Where files with extension `.h` are headers, `.a` are static libraries. The package is built on Ubuntu-24.04, thus follows UNIX naming convention.

### Import generated packages

Make a demo project named `Baz` with layout:
```
.
├── CMakeLists.txt
├── deps
│   ├── googletest-1.15.2
│   └── out
└── test_baz.cc
```

Where `googletest-1.15.2` is downloaded and unpacked at [GoogleTest's release page on github](https://github.com/google/googletest/releases/tag/v1.15.2).

And the `CMakeLists.txt`:
```
cmake_minimum_required(VERSION 3.28)
set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

project(Baz)

set(GTest_DIR ${CMAKE_CURRENT_LIST_DIR}/deps/out/${CMAKE_BUILD_TYPE}/lib/cmake/GTest)
find_package(GTest REQUIRED)
add_executable(Baz test_baz.cc)
target_link_libraries(Baz PRIVATE GTest::gmock_main)
if (WIN32)
  set_property(TARGET Baz PROPERTY MSVC_RUNTIME_LIBRARY "MultiThreaded")
endif()
```

The executable source code:
```
// test_baz.cc
#include <string>
#include <vector>

#include "gmock/gmock.h"
using ::testing::ElementsAre;

TEST(Baz, basic) {
  std::vector<std::string> v = {"foo","bar","baz"};
  EXPECT_THAT(v, ElementsAre("foo", "bar", "baz"));
}
```

Build with:
```
cmake -B build -S . -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH=deps/out/Release
cmake --build build --config Release
```

Execute `./build/Baz`:
```
Running main() from gmock_main.cc
[==========] Running 1 test from 1 test suite.
[----------] Global test environment set-up.
[----------] 1 test from Baz
[ RUN      ] Baz.basic
[       OK ] Baz.basic (0 ms)
[----------] 1 test from Baz (0 ms total)

[----------] Global test environment tear-down
[==========] 1 test from 1 test suite ran. (0 ms total)
[  PASSED  ] 1 test.
```

### Make `find_package` work

CMake's `find_package` hides many magics behind and tells you basically nothing when it fails:
```
CMake Error at /usr/share/cmake-3.28/Modules/FindPackageHandleStandardArgs.cmake:230 (message):
  Could NOT find GTest (missing: GTEST_LIBRARY GTEST_INCLUDE_DIR
  GTEST_MAIN_LIBRARY)
Call Stack (most recent call first):
  /usr/share/cmake-3.28/Modules/FindPackageHandleStandardArgs.cmake:600 (_FPHSA_FAILURE_MESSAGE)
  /usr/share/cmake-3.28/Modules/FindGTest.cmake:270 (FIND_PACKAGE_HANDLE_STANDARD_ARGS)
  CMakeLists.txt:8 (find_package)
```

But fortunately, its [document](https://cmake.org/cmake/help/latest/command/find_package.html) is quite comprehensive. With the document's help, we know the `find_package` is searching for files naming like `Find<PackageName>.cmake` or `<lowercasePackageName>-config.cmake` or `<PackageName>Config.cmake`. For the demo project, GoogleTest's config file locates at `deps/out/Release/lib/cmake/GTest`, CMake needs to know this path and there are 2 ways to reach that.

1. Throuth variable `GTest_DIR`

    Set `<PackageName>_DIR` before `find_package`, which is the directory contains `<PackageName>Config.cmake`. For GoogleTest, add `set(GTest_DIR ${CMAKE_CURRENT_LIST_DIR}/deps/out/${CMAKE_BUILD_TYPE}/lib/cmake/GTest)` in CMakeLists.txt. Use build-in variables like `CMAKE_CURRENT_LIST_DIR` and `CMAKE_BUILD_TYPE` to make it more flexible.

2. Throuth configuration `-DCMAKE_PREFIX_PATH`

    Add `-DCMAKE_PREFIX_PATH=deps/out/Release` at configuration stage, then CMake searchs [a list of directories](https://cmake.org/cmake/help/latest/command/find_package.html#config-mode-search-procedure) and would reach the `GTest`.

> I listed both of them in the demo project to make it clear, choose one of them in practice.

## Common issues

### `find_package` fails

I think this is the hardest one. Check the `<PackageName>Config.cmake` or `CMAKE_PREFIX_PATH` again. Also try to use [`message`](https://cmake.org/cmake/help/latest/command/message.html) to print out them.

It may continue raising errors after adding them. For example, I encountered following issue when importing [raylib](https://github.com/raysan5/raylib) on windows.

```Could NOT find raylib (missing: raylib_LIBRARY raylib_INCLUDE_DIR)```

It gone after adding settings:
```
set(raylib_INCLUDE_DIR ${<PROJ_NAME>_DEPS_OUTPUT_DIR}/include)
set(raylib_LIBRARY ${<PROJ_NAME>_DEPS_OUTPUT_DIR}/lib/raylib.lib)
```

> On ubuntu, such thing is not required and it works fine.

### Link issues

- `error LNK2038: mismatch detected for 'RuntimeLibrary': value 'MT_StaticRelease' doesn't match value 'MD_DynamicRelease'`

    MSVC use [`/MT[d]` and `/MD[d]`](https://learn.microsoft.com/en-us/cpp/build/reference/md-mt-ld-use-run-time-library) to select runtime library at compile stage. The link error raises when some code is compiled with a different setting with other codes. Add `-v` to the build command to see what arguments are passed to the compiler.

- `warning D9025 : overriding '/MD' with '/MT'`

    Similary to the one above, use `-v` to show compile commands and to debug.
