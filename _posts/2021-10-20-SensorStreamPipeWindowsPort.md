---
layout: post
title: "Sensor-Stream-Pipe: a Windows port, some SWIG, and Azure Kinect"
---

Contributions to [moetsi/Sensor-Stream-Pipe](https://github.com/moetsi/Sensor-Stream-Pipe) from Aug to Nov 2021, mostly on the `raas` branch. SSP is an open-source C++ pipeline for compressing and streaming RGB-D sensor data — Azure Kinect, OAK-D, iOS ARKit — to a remote server. Built on FFmpeg / LibAV + NVCodec.

## What I worked on

Windows support was most of it. SSP had grown up on Linux and macOS; making it build and run under MSVC / Cygwin on Windows exposed a pile of the usual suspects:

- **SWIG / C# name conflicts** — generating C# bindings from the C++ headers hit name collisions that MSVC's mangling + SWIG's wrapper generator didn't catch the same way as GCC would. `SWIG/C# name conflict` commit is the reminder.
- **Cygwin `unzip` chain** — the build pulled external deps via `unzip` that Cygwin didn't ship by default. Tried `jar` (Java's unzip), which was "too intelligent" about file paths. Ended up pinning a specific Windows unzip.
- **CMake / Windows** — the CMakeLists needed Windows-specific find-module chains. `cmake --help for debug` stayed in a commit longer than it should have.
- **Azure Kinect (`k4a`)** — linking the Azure Kinect SDK on Windows; separate paths for the dev libs vs the runtime redistributables.
- **Memory leaks** — `delete on malloc`, `leaksgit add -A git add -A` (the commit message literally). A batch of the usual leaks in pre-RAII code paths.
- **Network byte order** — `inplace_hton` for wire-format correctness across big/little-endian hosts.

## The RAAS branch

RAAS (the "Remote Assistance as a Service" shape — the branch name) pulled in the compressed-frame streaming with a heartbeat, throttle control, and a body-reader. Most of the commits from me on this branch were integration glue: merging main into RAAS repeatedly, fixing compile breaks across the merge, then re-fixing when the other branch moved.

## What I learned

A cross-platform C++ project that has to ship on MSVC is a different project from one that only has to build on GCC / Clang. The difference shows up in CMake toolchain files, dependency managers (vcpkg vs Homebrew vs apt), binding generators (SWIG's Windows path), and in the precise kind of warnings each compiler treats as errors. Two years later the ecosystem had settled a bit; in 2021 it was still a manual job.

Contributions @ [moetsi/Sensor-Stream-Pipe](https://github.com/moetsi/Sensor-Stream-Pipe).
