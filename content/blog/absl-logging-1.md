+++
pubdate = '2025-11-30'
lastmod = '2025-12-01'
draft = false
title = 'Abseil Logging with Meson Build - Episode 1'
+++

![absl-meson](/assets/absl-logging/absl-meson.jpg)

In this post, we are going to talk about how to use `Meson` to incorparate
`Abseil` Logging library into our C++ projects.

## What is Abseil

"Abseil is an open source collection of C++ libraries drawn from the most
fundamental pieces of Google’s internal codebase", according to the offical
Abseil introduction. Shortly speaking, it supports C++ code with enhanced
features by providing missing pieces from the C++ standard and helpful utilities
built in the Google world.

The Abseil Logging Library is considered as the successor of the public archived
Google Logging Library (glog).

## Why using Meson

Meson Build system is considered to be more readable and user friendly than
CMake in most cases. I personally attribute this to two reasons:

- Instead of exposing thousands of low-level interfaces, Meson adopt the
  `dependency` abstraction without losing many helpful utilities.
- Meson is even more powerful when we handle multi-platform development while
  managing dependencies, using its own `WrapDB` features.

## Create a Meson Template

Let's start by creating a Meson project with empty files.

```bash
$ mkdir absl-logging && cd absl-logging

$ mkdir -p \
    apps/AbslLogExample \
    include/AbslLogger \
    lib/SingletonAbslLogger

$ touch \
    meson.build \
    apps/AbslLogExample/AbslLogExample.cc \
    include/AbslLogger/AbslLogger.hh \
    lib/SingletonAbslLogger/SingletonAbslLogger.hh \
    lib/SingletonAbslLogger/SingletonAbslLogger.cc

$ tree
absl-logging/
├── apps
│   └── AbslLogExample
│       └── AbslLogExample.cc
├── include
│   └── AbslLogger
│       └── AbslLogger.hh
├── lib
│   └── SingletonAbslLogger
│       ├── SingletonAbslLogger.cc
│       └── SingletonAbslLogger.hh
└── meson.build
```

According to the official Abseil tutorial, we are now fetching the source code
of Abseil as a third-party library. This is incredibly simple using Meson:

```bash
$ mkdir subprojects

$ meson wrap list | rg abseil
abseil-cpp

$ meson wrap install abseil-cpp
Installed abseil-cpp version 20250814.1 revision 1

$ ls subprojects/
abseil-cpp.wrap

$ cat subprojects/abseil-cpp.wrap | head --lines=20
[wrap-file]
directory = abseil-cpp-20250814.1
source_url = https://github.com/abseil/abseil-cpp/releases/download/20250814.1/abseil-cpp-20250814.1.tar.gz
source_filename = abseil-cpp-20250814.1.tar.gz
source_hash = 1692f77d1739bacf3f94337188b78583cf09bab7e420d2dc6c5605a4f86785a1
patch_filename = abseil-cpp_20250814.1-1_patch.zip
patch_url = https://wrapdb.mesonbuild.com/v2/abseil-cpp_20250814.1-1/get_patch
patch_hash = f44ad4d72ff2919a6a48cf0887f69aef34c338bfdd8931153596e3766d75f654
source_fallback_url = https://github.com/mesonbuild/wrapdb/releases/download/abseil-cpp_20250814.1-1/abseil-cpp-20250814.1.tar.gz
wrapdb_version = 20250814.1-1

[provide]
absl_base = absl_base_dep
absl_container = absl_container_dep
absl_debugging = absl_debugging_dep
absl_log = absl_log_dep
absl_flags = absl_flags_dep
absl_hash = absl_hash_dep
absl_crc = absl_crc_dep
absl_numeric = absl_numeric_dep
```

What's going on here is that Meson create a wrap file under `subprojects`
directory which records metadata. It will be then used for fetching the source
code when compilation is triggered the first time.

Ok! Time for codes!

## A naive Abseil Singleton Logger

First of all, we need to have some tool interfaces:

```cpp
/* --- include/AbslLogger/AbslLogger.hh --- */
#pragma once
#include <cstdint>
#include <string_view>

enum class LogLevel : std::uint8_t {
  INFO,
  WARNING,
  ERROR,
};

class AbslLogger {
public:
  static auto setLevel(LogLevel lvl) -> void;
  static auto info(std::string_view msg) -> void;
  static auto warning(std::string_view msg) -> void;
  static auto error(std::string_view msg) -> void;
};
```

And a toy example:

```cpp
/* --- apps/AsblLogExample/AbslLogExample.cpp --- */
#include "AbslLogger/AbslLogger.hh"

auto main() -> int {
  AbslLogger::setLevel(LogLevel::INFO);
  AbslLogger::info("Direct static INFO log");
  AbslLogger::warning("Direct static WARNING log");
  AbslLogger::error("Direct static ERROR log");
  return 0;
}
```

We then continue our toy thread-safe singleton logger like this:

```cpp
/* --- lib/SingletonAbslLogger/SingletonAbslLogger.hh --- */
#pragma once
#include <cstdint>
#include <string_view>
template <typename T>

class Singleton_ {
public:
  Singleton_() = delete;
  ~Singleton_() = delete;
  Singleton_(const Singleton_ &) = delete;
  Singleton_(Singleton_ &&) = delete;
  auto operator=(const Singleton_ &) -> Singleton_ & = delete;
  auto operator=(Singleton_ &&) -> Singleton_ & = delete;
  static auto instance() -> T & {
    static T inst;
    return inst;
  }
};

enum class LogLevel : std::uint8_t {
  INFO,
  WARNING,
  ERROR,
};

class LoggerImpl_ {
public:
  LoggerImpl_();
  auto setLevel(LogLevel lvl) -> void;
  auto info(std::string_view msg) const -> void;
  auto warning(std::string_view msg) const -> void;
  auto error(std::string_view msg) const -> void;

private:
  LogLevel LoggerLevel = LogLevel::WARNING;
};

class AbslLogger {
  static auto setLevel(LogLevel lvl) -> void;
  static auto info(std::string_view msg) -> void;
  static auto warning(std::string_view msg) -> void;
  static auto error(std::string_view msg) -> void;
};


/* --- lib/SingletonAbslLogger/SingletonAbslLogger.cc --- */
#include "SingletonAbslLogger.hh"
#include "absl/base/log_severity.h"
#include "absl/log/absl_log.h"
#include "absl/log/globals.h"
#include "absl/log/initialize.h"
#include <string_view>

LoggerImpl_::LoggerImpl_() {
  absl::InitializeLog();
  absl::SetStderrThreshold(absl::LogSeverity::kInfo);
}

auto LoggerImpl_::setLevel(LogLevel lvl) -> void { LoggerLevel = lvl; }

auto LoggerImpl_::info(std::string_view msg) const -> void {
  ABSL_LOG_IF(INFO, LoggerLevel <= LogLevel::INFO) << msg;
}

auto LoggerImpl_::warning(std::string_view msg) const -> void {
  ABSL_LOG_IF(WARNING, LoggerLevel <= LogLevel::WARNING) << msg;
}
auto LoggerImpl_::error(std::string_view msg) const -> void {
  ABSL_LOG_IF(ERROR, LoggerLevel <= LogLevel::ERROR) << msg;
}

auto AbslLogger::setLevel(LogLevel lvl) -> void {
  Singleton_<LoggerImpl_>::instance().setLevel(lvl);
}

auto AbslLogger::info(std::string_view msg) -> void {
  Singleton_<LoggerImpl_>::instance().info(msg);
}

auto AbslLogger::warning(std::string_view msg) -> void {
  Singleton_<LoggerImpl_>::instance().warning(msg);
}

auto AbslLogger::error(std::string_view msg) -> void {
  Singleton_<LoggerImpl_>::instance().error(msg);
}
```

And a `meson.build` file to combine all together:

```meson
## --- meson.build --- ##
project('absl-logging', 'cpp')

liblog = shared_library(
  'log',
  ['lib/SingletonAbslLogger/SingletonAbslLogger.cc'],
  include_directories: [
    'lib/SingletonAbslLogger',
  ],
  dependencies: [
    dependency('absl_log'),
  ]
)

log_dep = declare_dependency(
  link_with: liblog,
  include_directories: [
    'include',
  ]
)

executable('AbslLogExample',
  'apps/AbslLogExample/AbslLogExample.cc',
  dependencies: [
    log_dep,
  ],
)
```

Now build:

```bash
$ meson setup build
The Meson build system
Version: 1.9.1
Source dir: /home/amas/Projects/absl-logging
Build dir: /home/amas/Projects/absl-logging/build
Build type: native build
Project name: absl-logging
Project version: undefined
C++ compiler for the host machine: clang++ (clang 21.1.6 "clang version 21.1.6 (Fedora 21.1.6-1.fc43)")
C++ linker for the host machine: clang++ ld.bfd 2.45.1-1
Host machine cpu family: x86_64
Host machine cpu: x86_64
Found pkg-config: YES (/usr/bin/pkg-config) 2.3.0
Found CMake: /usr/bin/cmake (3.31.6)
Run-time dependency absl_log found: NO (tried pkgconfig and cmake)
Looking for a fallback subproject for the dependency absl_log
Downloading abseil-cpp source from https://github.com/abseil/abseil-cpp/releases/download/20250814.1/abseil-cpp-20250814.1.tar.gz
Download size: 2235716
Downloading: ..........
Downloading abseil-cpp patch from https://wrapdb.mesonbuild.com/v2/abseil-cpp_20250814.1-1/get_patch
Download size: 6092
Downloading: ..........

Executing subproject abseil-cpp

abseil-cpp| Project name: abseil-cpp
abseil-cpp| Project version: 20250814.1
abseil-cpp| C++ compiler for the host machine: clang++ (clang 21.1.6 "clang version 21.1.6 (Fedora 21.1.6-1.fc43)")
abseil-cpp| C++ linker for the host machine: clang++ ld.bfd 2.45.1-1
abseil-cpp| Compiler for C++ supports arguments /DNOMINMAX: NO
abseil-cpp| Compiler for C++ supports arguments -Wcast-qual: YES
abseil-cpp| Compiler for C++ supports arguments -Wconversion-null: YES
abseil-cpp| Compiler for C++ supports arguments -Wmissing-declarations: YES
abseil-cpp| Compiler for C++ supports arguments -Wno-comma: YES
abseil-cpp| Compiler for C++ supports arguments -Wno-conversion: YES
abseil-cpp| Compiler for C++ supports arguments -Wno-double-promotion: YES
abseil-cpp| Compiler for C++ supports arguments -Wno-float-promotion: NO
abseil-cpp| Compiler for C++ supports arguments -Wno-format-literal: NO
abseil-cpp| Compiler for C++ supports arguments -Wno-gcc-compat: YES
abseil-cpp| Compiler for C++ supports arguments -Wno-old-style-cast: YES
abseil-cpp| Compiler for C++ supports arguments -Wno-packed: YES
abseil-cpp| Compiler for C++ supports arguments -Wno-padded: YES
abseil-cpp| Compiler for C++ supports arguments -Wno-pedantic: YES
abseil-cpp| Compiler for C++ supports arguments -Wno-range-loop-analysis: YES
abseil-cpp| Compiler for C++ supports arguments -Wno-sign-compare: YES
abseil-cpp| Compiler for C++ supports arguments -Woverlength-strings: YES
abseil-cpp| Compiler for C++ supports arguments -Wpointer-arith: YES
abseil-cpp| Compiler for C++ supports arguments -Wswitch-enums: NO
abseil-cpp| Compiler for C++ supports arguments -Wunused-local-typedefs: YES
abseil-cpp| Compiler for C++ supports arguments -Wunused-result: YES
abseil-cpp| Compiler for C++ supports arguments -Wvarargs: YES
abseil-cpp| Compiler for C++ supports arguments -Wvla: YES
abseil-cpp| Compiler for C++ supports arguments -Wwrite-strings: YES
abseil-cpp| Compiler for C++ supports arguments -maes: YES
abseil-cpp| Compiler for C++ supports arguments -msse4.1: YES
abseil-cpp| Checking if "atomic builtins" links: YES
abseil-cpp| Run-time dependency threads found: YES
abseil-cpp| Run-time dependency appleframeworks found: NO (tried framework)
abseil-cpp| Build targets in project: 14
abseil-cpp| Subproject abseil-cpp finished.

Dependency absl_log from subproject subprojects/abseil-cpp-20250814.1 found: YES 20250814.1
Build targets in project: 16

absl-logging undefined

  Subprojects
    abseil-cpp: YES

Found ninja-1.13.1 at /usr/bin/ninja

$ meson compile -C build/
INFO: autodetecting backend as ninja
INFO: calculating backend command to run: /usr/bin/ninja -C /home/amas/Projects/absl-logging/build
ninja: Entering directory `/home/amas/Projects/absl-logging/build'
[171/171] Linking target AbslLogExample
```

Here we go:

```bash
$ ./build/AbslLogExample
I1130 22:19:21.712713   29152 SingletonAbslLogger.cc:16] Direct static INFO log
W1130 22:19:21.713893   29152 SingletonAbslLogger.cc:20] Direct static WARNING log
E1130 22:19:21.713912   29152 SingletonAbslLogger.cc:23] Direct static ERROR log
```

## Change Logger Level

It turns out that we only need to **include this header file** and **link the
library** in order to output some logs during the whole program life. And we can
leave all `Abseil` and `Singleton` magics or issues from the main part. What an
ease!

We can also raise the "LoggerLevel" to filter out over-detailed debugging/info
messages. By doing this, we can focus on important stuffs without being
overwhelmed.

```bash
$ git diff
diff --git a/apps/AbslLogExample/AbslLogExample.cc b/apps/AbslLogExample/AbslLogExample.cc
index d14e6f4..170f5bf 100644
--- a/apps/AbslLogExample/AbslLogExample.cc
+++ b/apps/AbslLogExample/AbslLogExample.cc
@@ -1,7 +1,7 @@
  1 ⋮  1 │ #include "AbslLogger/AbslLogger.hh"
  2 ⋮  2 │
  3 ⋮  3 │ auto main() -> int {
  4 ⋮    │-  AbslLogger::setLevel(LogLevel::INFO);
    ⋮  4 │+  AbslLogger::setLevel(LogLevel::WARNING);
  5 ⋮  5 │   AbslLogger::info("Direct static INFO log");
  6 ⋮  6 │   AbslLogger::warning("Direct static WARNING log");
  7 ⋮  7 │   AbslLogger::error("Direct static ERROR log");

$ meson compile -C build/
INFO: autodetecting backend as ninja
INFO: calculating backend command to run: /usr/bin/ninja -C /home/amas/Projects/absl-logging/build
ninja: Entering directory `/home/amas/Projects/absl-logging/build'
[2/2] Linking target AbslLogExample

$ ./build/AbslLogExample
W1130 22:44:26.754305   34373 SingletonAbslLogger.cc:20] Direct static WARNING log
E1130 22:44:26.755098   34373 SingletonAbslLogger.cc:23] Direct static ERROR log
```

## Conclusion

In this post, we create a "Singleton Logger" based on `Abseil` using `Meson`.
With a built-in logging feature, we are able to filter important log messages
and find out issues quickly when exceptions are raised or during debugging.
