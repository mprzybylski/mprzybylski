---
layout: post
title: "Packaging libbpf with vcpkg"
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
  - packaging
  - "dependency management"
date: 2024-05-09 09:00 -0800
---
![Box in a box](/images/box_in_a_box.JPG)
# Previously
[Managing dependencies with vcpkg]({% post_url 2024-05-13-Managing-dependencies-with-vcpkg %})
# `libbpf`: the missing package
In the interest of maintainability, I want `vcpkg` to be `bpf-iotrace`'s one-stop shop for managing third-party code.
Since vcpkg's default registry doesn't include `libbpf` I will need to package it the vcpkg way, myself.

I'll make an educated guess that the biggest reason libbpf isn't already in the default `vcpkg` registry is that its
build procedure diverges pretty sharply from what is typical for a Makefile-based open source project.
Most Makefile-based open source projects use
[GNU autotools](https://www.gnu.org/software/automake/manual/html_node/Autotools-Introduction.html)
to generate a Makefile tailored for the build environment and the target system.
Those projects also typically separate the source and build directories, also known as an out-of-source build.
`libbpf` doesn't require this level of complexity
and uses a [manually written Makefile](https://github.com/libbpf/libbpf/blob/master/src/Makefile)
located in the same directory as the library source.
Also, `make` must be called from the source directory, (an in-source build).
This has the advantage of fewer dependencies and faster build times.
The disadvantage is that
[`vcpkg_configure_make()`](https://learn.microsoft.com/en-us/vcpkg/maintainers/functions/vcpkg_configure_make) and
[`vcpkg_build_make()`](https://learn.microsoft.com/en-us/vcpkg/maintainers/functions/vcpkg_configure_make) aren't
written to work with a project that is structured this way.

# Adapting the `libbpf` build to be compatible with `vcpkg` conventions

As I noted in [my previous article]({% post_url 2024-05-13-Managing-dependencies-with-vcpkg %}),
an [overlay ports](https://learn.microsoft.com/en-us/vcpkg/concepts/overlay-ports) directory is an excellent
place to keep a library while developing and testing because it enables rapid iteration.
So I created the `tools/vcpkg-overlays/ports/libbpf` and its parents in the `bpf-iotrace` project and referenced
`tools/vcpkg-overlays/ports` in
[`vcpkg-configuration.json`](https://github.com/mprzybylski/bpf-iotrace/blob/main/vcpkg-configuration.json).
Then I added the following files to `tools/vcpkg-overlays/ports/libbpf`:

* [`vcpkg.json]`(https://github.com/mprzybylski/any-port-in-a-storm/blob/main/ports/libbpf/vcpkg.json): 
  The manifest file that describes the port and declares `libbpf`'s direct dependency on
  [elfutils libelf](https://sourceware.org/elfutils/).

* [`CMakeLists.txt`](https://github.com/mprzybylski/any-port-in-a-storm/blob/main/ports/libbpf/CMakeLists.txt):
  This file wraps the `libbpf` build system to make it look from the outside like a typical CMake project.
  `vcpkg` works really easily with CMake projects.

  `vcpkg` ports like `bzip2` use a
  [`CMakeLists.txt` file that completely bypasses the upstream project's build system](https://github.com/microsoft/vcpkg/blob/master/ports/bzip2/CMakeLists.txt).
  While this may be a good fit for projects that rarely change, it is not a good fit for `libbpf`,
  which is constantly evolving.  It is highly likely that new releases of `libbpf` would break a `CMakeLists.txt`
  that attempts to bypass the existing `Makefile` by adding new source files or refactoring existing ones.
  Keeping those parallel build systems in sync would simply be more trouble than it's worth.

  Fortunately, CMake has a module called
  [`ExternalProject`](https://cmake.org/cmake/help/latest/module/ExternalProject.html)
  that allows a CMake project to wrap project builds involving non-CMake build systems.
  Even more fortunately, `libbpf`'s Makefile follows
  [`GNU Makefile conventions`](https://www.gnu.org/prep/standards/html_node/Makefile-Conventions.html)
  which give me a stable interface to use when wrapping libbpf's build with `ExternalProject_Add()`.

  `CMakeLists.txt` also translates the "implicit options" `vcpkg` passes via
  [`vcpkg_cmake_configure()`](https://learn.microsoft.com/en-us/vcpkg/maintainers/functions/vcpkg_cmake_configure)
  to Makefile variables and installation paths.

* [`portfile.cmake`](https://github.com/mprzybylski/any-port-in-a-storm/blob/main/ports/libbpf/portfile.cmake) that
  fetches and unpacks the specified version of `libbpf`, calls `vcpkg_cmake_configure()`,
  [`vcpkg_cmake_build()`](https://learn.microsoft.com/en-us/vcpkg/maintainers/functions/vcpkg_cmake_build), and
  [`vcpkg_cmake_install()`](https://learn.microsoft.com/en-us/vcpkg/maintainers/functions/vcpkg_cmake_install) to
  build and install `libbpf`.

# Brushing off the lint
After my first successful build of `libbpf`, in `vcpkg`, I noticed the following warnings in CMake's output:
```text
...
-- Performing post-build validation
warning: The software license must be available at ${CURRENT_PACKAGES_DIR}/share/libbpf/copyright

    vcpkg_install_copyright(FILE_LIST "${SOURCE_PATH}//IdeaProjects/bpf-iotrace/tools/vcpkg/buildtrees/libbpf//LICENSE")
warning: There should be no absolute paths, such as the following, in an installed package:
  /IdeaProjects/bpf-iotrace/tools/vcpkg/packages/libbpf_x64-linux
  /IdeaProjects/bpf-iotrace/cmake-build-debug/vcpkg_installed
  /IdeaProjects/bpf-iotrace/tools/vcpkg/buildtrees/libbpf
  /IdeaProjects/bpf-iotrace/tools/vcpkg/downloads
Absolute paths were found in the following files:
  /IdeaProjects/bpf-iotrace/tools/vcpkg/packages/libbpf_x64-linux/debug/lib/pkgconfig/libbpf.pc
  /IdeaProjects/bpf-iotrace/tools/vcpkg/packages/libbpf_x64-linux/lib/pkgconfig/libbpf.pc
error: Found 2 post-build check problem(s). To submit these ports to curated catalogs, please first correct the portfile: /IdeaProjects/bpf-iotrace/./tools/vcpkg-overlays/ports/libbpf/portfile.cmake
...
```

Thankfully, `vcpkg` provides a pair of functions I can call from `portfile.cmake` to resolve these issues easily:
* [`vcpkg_install_copyright()`](https://learn.microsoft.com/en-us/vcpkg/maintainers/functions/vcpkg_install_copyright)
takes a list of paths to license files and legal notices for the upstream project, and places them in `vcpkg`'s
preferred location.
* [`vcpkg_fixup_pkgconfig()`](https://learn.microsoft.com/en-us/vcpkg/maintainers/functions/vcpkg_fixup_pkgconfig)
transforms any absolute paths in a library's [pkg-config](https://www.freedesktop.org/wiki/Software/pkg-config/)
file to relative paths which are preferred by `vcpkg`.

# Graduating from the overlay directory
Once a library has been packaged successfully, it can be graduated to a public or private
[registry](https://learn.microsoft.com/en-us/vcpkg/maintainers/registries) to facilitate its use in other projects.

Currently, only an empty `.gitignore` file exists in `bpf-iotrace/tools/vcpkg-overlays/ports` to make sure the directory
gets created by `git clone ...`.
However, I will still temporarily copy in
[`any-port-in-a-storm/ports/libbpf`](https://github.com/mprzybylski/any-port-in-a-storm/blob/main/ports/libbpf) when
I want to test changes or fixes my `libbpf` `vcpkg` port.

# Up next
eBPF programs the CMake way