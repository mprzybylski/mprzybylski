---
layout: post
title: "eBPF programs the CMake way"
category: libbpf-based tracing from scratch
tags: 
  - Linux
  - software
  - engineering
  - BPF
  - eBPF
  - development
  - vcpkg
  - make
  - CMake
  - clang
  - bpftool
# Watch out for timestamps that are the next day, UTC
date: 2024-05-17 09:00 -0800
---
![eBPF / CMake logo mashup](/images/ebpf-the-cmake-way.png)

# Previously
[Packaging libbpf with vcpkg]({% post_url 2024-05-15-Packaging-libbpf-with-vcpkg %})

# An unusual library
It is highly desirable for eBPF code to look like just another library in a CMake project.
This makes it easier for new developers to figure out what's going on with the build, and
enables advanced code navigation features in IDEs like [CLion](https://www.jetbrains.com/clion/).
However, there are enough differences between the way a native library is built
and the way an eBPF object file needs to be built that CMake's 
[add_library()](https://cmake.org/cmake/help/latest/command/add_library.html) 
command needs to be used a bit differently.

# Prerequisites
As noted in
[Creating a build environment for libbpf-based programs]({% post_url 2024-05-09-Creating-a-build-environment-for-libbpf-based-programs %}),
[Clang](https://clang.llvm.org/) and [bpftool](https://github.com/libbpf/bpftool)
are essential for producing eBPF object files.
The rest of this article assumes they are installed on the build host or container.

# Start with a dedicated subdirectory and `CMakeLists.txt`
To produce an eBPF object file from a CMake "library" target,
the `CMAKE_C_FLAGS` variable must be overridden and set to `-target bpf -Werror -g -O2`.
The `CMAKE_C_COMPILER` variable will also need to be overridden unless the outer project's C
code is already being built with `clang`.
A dedicated subdirectory or subtree for eBPF source code takes advantage of
[CMake variable scoping rules](https://cmake.org/cmake/help/book/mastering-cmake/chapter/Writing%20CMakeLists%20Files.html#variable-scope)
to limit the effects of these overrides to just the locations where they are needed.

# eBPF C Flags explained
 - `-target bpf`: Generate eBPF code.
   _**NOTE:** eBPF follows the [endianness](https://en.wikipedia.org/wiki/Endianness#Overview) of the processor on
   the target system.
   `clang`'s `bpf` target follows the endianness of the build host's processor.
   If you are cross-compiling, make sure your build system has logic to select `bpfeb` (big endian),
   or `bpfel`, (little endian) based on the endianness of your target system._
 - `-Werror`: The [eBPF verifier](https://www.kernel.org/doc/html/latest/bpf/verifier.html) in older Linux kernels
   will reject programs with backward jumps, (loops).
   (Even in newer kernels, the verifier requires a loop to have a finite trip count.)
   The solution for eBPF programs written in C is to add `#pragma clang loop unroll(full)` just above the beginning
   of the loop statement to have the compiler unroll those loops.
   (See [Clang's documentation](https://clang.llvm.org/docs/LanguageExtensions.html#loop-unrolling) for more
   information.)
   If the compiler _fails_ to unroll a loop (usually because it can't determine a finite trip count from the loop's
   condition statement), it will generate a warning.
   For native code, such a warning is acceptable.
   For eBPF code, failure to unroll a loop is usually a fatal error.
   So `-Werror` causes the compiler to treat _all_ warnings as errors.
 - `-g`: Include source-level debug info in generated objects.
   Clang will add [DWARF](https://dwarfstd.org/) and [BTF](https://www.kernel.org/doc/html/latest/bpf/btf.html) 
   debugging data to its output files.
   DWARF is helpful for debugging eBPF verifier rejections, and BTF is required for libbpf to perform
   [CO-RE relocations](https://github.com/libbpf/libbpf#bpf-co-re-compile-once--run-everywhere).
 - `-O2`: Optimization level 2

# File organization
My favorite way to organize an eBPF project is by program entry point.
For system call tracepoints, this means one `.c` and `.h` file per system call, i.e. `read.bpf.c` and `read.bpf.h` or
`execve.c` and `execve.h`.
Entry and exit handler programs should live in the same source file.  Common helper functions should live in
separate header files.

# CMake targets
Trying to build eBPF code as a
["normal" library](https://cmake.org/cmake/help/latest/command/add_library.html#normal-libraries)
in CMake will fail at the linking step because eBPF object files don't link like native code.

To build eBPF code with a CMake target, we have to use the
[object library](https://cmake.org/cmake/help/latest/command/add_library.html#object-libraries)
signature of `add_library()`

There should be one CMake target per C file, and the C file and the target should have the same base name,
i.e. `read.bpf.c` is built by a CMake target called `read.bpf`:

```
add_library(read OBJECT read.bpf.c read.bpf.h)
```

This passes `read.bpf.c` to `clang -c` which compiles and assembles the file, but stops short of linking.

To do the equivalent of linking an eBPF project,
we need to write some explicit post-processing steps into our `CMakeLists.txt`.

We can reference the output(s) of an object library target with the following CMake
[generator expression](https://cmake.org/cmake/help/latest/manual/cmake-generator-expressions.7.html):
```
$<TARGET_OBJECTS:target_name>
```

This allows us to pass them to
[add_custom_command()](https://cmake.org/cmake/help/latest/command/add_custom_command.html):

```
add_custom_command(OUTPUT combined_bpf_lib.o
        COMMAND bpftool gen object combined_bpf_lib.o $<TARGET_OBJECTS:target_a> $<TARGET_OBJECTS:target_b>
        DEPENDS $<TARGET_OBJECTS:target_a> $<TARGET_OBJECTS:target_b>
        VERBATIM)
```

"[bpftool gen object ...](https://github.com/libbpf/bpftool/blob/main/docs/bpftool-gen.rst)"
takes the `.o` files generated by `target_a` and `target_b`, combines them in a manner similar to the way
`ld` would statically link user-space object files, and de-duplicates their BTF data.
The output file is `combined_bpf_lib.o`.

We can also go a step further and package `combined_bpf_lib.o` for embedding in a C or C++ binary with:
```
add_custom_command(OUTPUT bpf-skeleton.h
        COMMAND bpftool gen skeleton combined_bpf_lib.o > bpf-skeleton.h
        DEPENDS combined_bpf_lib.o
        VERBATIM)
```

Finally, we can create a named target that depends on these custom commands:
```
add_custom_target(bpf_skeleton DEPENDS bpf-skeleton.h)
```

# A DRY (Don't Repeat Yourself) CMakeLists.txt
Most eBPF projects are going to implement several eBPF programs.
Each C file containing an eBPF program will require adding an...
```
add_library(... OBJECT ...)
```
...line to `CMakeLists.txt` and editing...
```
add_custom_command(OUTPUT combined_bpf_lib.o ...)
```
...to add a reference to its target objet files.
This gets unwieldy with a large number of eBPF programs.

A cleaner solution would be to create and update a list of target names with corresponding source files...
```
list(APPEND BPF_OBJECT_TARGETS open close read write readv writev pwrite64 pwritev pwritev2)
```
...and iterate over that list to create object library targets and a list of output files for post-processing.
```
foreach (TARGET IN LISTS BPF_OBJECT_TARGETS)
    add_library(${TARGET} OBJECT ${TARGET}.c ${TARGET}.h)
    list(APPEND BPF_OBJECT_LIBS $<TARGET_OBJECTS:${TARGET}>)
endforeach ()
```

This greatly simplifies the custom command for the linking step:
```
add_custom_command(OUTPUT combined_bpf_lib.o
        COMMAND bpftool gen object combined_bpf_lib.o ${BPF_OBJECT_LIBS}
        DEPENDS ${BPF_OBJECT_LIBS}
        VERBATIM)
```

# Go forth and code
Once this scaffolding is set up around your eBPF code, it should be easy to add new eBPF programs to your project
in a maintainable way.

# Up next
[A simple pair of eBPF tracepoint handlers]({% post_url 2024-06-17-A-simple-pair-of-eBPF-tracepoint-handlers %})