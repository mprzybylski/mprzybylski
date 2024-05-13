---
layout: post
title: "Creating a Build Environment for libbpf-based Programs"
category: libbpf-based tracing from scratch
tags: 
  - Linux
  - software
  - engineering
  - BPF
  - eBPF
  - development
  - gentoo
  - ebuild
  - portage
  - glibc
  - Docker
  - gcc
  - llvm
  - clang
  - "shared libraries"
  - "DLL hell"
  - "symbol versioning"
date: 2024-05-09 09:00 -0800
---
# Previously
[bpf-iotrace: Defining Requirements]({% post_url 2023-02-12-bpf-iotrace__Defining_Requirements %})

# A Boulevard of Broken Distros[^1]

Anyone who has used Linux for any length of time has probably seen an error message like this:
```
root@7e7960fedc17:/# nginx 
nginx: /lib/x86_64-linux-gnu/libcrypt.so.1: version `XCRYPT_2.0' not found (required by nginx)
nginx: /usr/lib/x86_64-linux-gnu/libssl.so.1.1: version `OPENSSL_1_1_1' not found (required by nginx)
nginx: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.27' not found (required by nginx)
nginx: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.28' not found (required by nginx)
```

These errors occur because when a binary is linked against a shared library, those links may be _versioned_.  If the shared library functions, (symbols), our application was linked against are newer than those available in the shared libraries on the system where our application is running now, we will see errors like the ones above.

So how do we keep this from happening to our projects?

[Statically linking a binary to GNU libc is strongly discouraged](https://stackoverflow.com/questions/57476533/why-is-statically-linking-glibc-discouraged) due to licensing issues and the tight coupling between GNU libc and filesystem layout choices in the Linux distributions where that binary will run.

One common way to make sure a binary works across a whole range of Linux distributions
and versions is to build it on the oldest distribution version a developer wants to support.
Generally, the shared libraries a binary depends on 
[contain additional symbols for backward compatibility](https://developers.redhat.com/blog/2019/08/01/how-the-gnu-c-library-handles-backward-compatibility),
so a binary built in this environment should just work, even on the latest and greatest supported distributions.[^2]

This seems to work well enough until a developer wants to use a newer C++ language standard than what is supported by the distro's bundled version of GCC and libstdc++.  A similar problem arises with clang/LLVM and libbpf/eBPF development: The latest libbpf features are only supported by the latest versions of `clang`.

One naive solution would be to try to install the GCC package from a newer version of the same linux distribution.  This almost worked when trying to install GCC 10.2 from Debian 11 in a Debian 9 container.  Unfortunately, the GCC 10.2 package has a transitive dependency on Debian 11's newer GNU libc package.  This either prevents GCC 10.2 from being installed, or forces Debian 9's GNU libc package to be upgraded.  This, in turn, thwarts the plan to link a project against the oldest GNU `libc` it can get away with.  I've also seen or experienced approaches like this completely breaking other distributions.  ["The only winning move is not to play."](https://www.youtube.com/watch?v=MpmGXeAtWUw)

In a previous job, I've solved the new-compiler-on-an-old-distro problem by building a newer version of GCC from source in a container.  However, there is another problem with building a project on an ancient distribution: It is more likely to have unpatched security vulnerabilities and/or grossly out-of-date OpenSSL, OpenSSH, etc.

# What about installing an old GNU libc in a newer Linux distribution just for development purposes?

[This almost works.](https://stackoverflow.com/a/52550158), but still has some issues due to how tightly coupled GCC is with the GNU libc it was built with.  If I want to minimize my chances of hitting subtle, hard-to-troubleshoot bugs, I will need to build the entire, tightly coupled toolchain.  This is not a trivial task to do manually, or to script from scratch.  Fortunately, Gentoo has a tool called [`crossdev`](https://gitweb.gentoo.org/proj/crossdev.git/tree/README). There is also the [crosstool-ng](https://crosstool-ng.github.io/) project.  Both utilities can automate the process of building a toolchain with an arbitrary compiler version, GNU libc version, kernel headers version, and target architecture.  I ended up choosing `crossdev` because it's easier to customize and drive completely as code, and introduces fewer layers of abstraction.  In short, it promises to be more maintainable.

# Choosing kernel headers and GNU libc versions.

eBPF gained enough features to start being truly useful around Linux kernel version 4.14.  So I wanted to build a toolchain that works with a vendor-supported Linux distribution with the oldest GNU libc and a kernel version >= 4.14.  The oldest combination of kernel and GNU libc I could find is Amazon Linux 2 with [GNU libc 2.26](https://docs.aws.amazon.com/AL2/latest/relnotes/relnotes.html#c-runtime) and [Linux kernel v4.14](https://docs.aws.amazon.com/linux/al2/ug/aml2-kernel.html).  (If you find a vendor-supported Linux distribution with Linux kernel >= 4.14 and an even older version of GNU libc, please file an [issue](https://github.com/mprzybylski/bpf-iotrace/issues), and let me know.)

# Trial and error with `crossdev` in docker

Armed with the kernel headers and glibc versions I needed to support, and the [crossdev documentation](https://wiki.gentoo.org/wiki/Crossdev), I set about creating a gentoo docker container and attempting to create my toolchain...

```shell
ad25027d037a / # crossdev --target x86_64-generic-linux-gnu --gcc '~14.1.0' --libc '~2.26' --kernel '~4.14' --ex-gdb
---------------------------------------------------------------------------------------
 * crossdev version:      20240209
 * Host Portage ARCH:     amd64
 * Host Portage System:   x86_64-pc-linux-gnu (i686-pc-linux-gnu x86_64-pc-linux-gnu)
 * Target Portage ARCH:   amd64
 * Target System:         x86_64-generic-linux-gnu
 * Stage:                 4 (C/C++ compiler)
 * USE=multilib:          no
 * Target ABIs:           amd64

 * binutils:              binutils-[latest]
 * gcc:                   ~gcc-14.1.0
 * headers:               ~linux-headers-4.14
 * libc:                  ~glibc-2.26
 * Extra: gdb:            DO IT

 * CROSSDEV_OVERLAY:      /var/db/repos/crossdev
 * PORT_LOGDIR:           /var/log/portage
 * PORTAGE_CONFIGROOT:    /
 * Portage flags:         
  _  -  ~  -  _  -  ~  -  _  -  ~  -  _  -  ~  -  _  -  ~  -  _  -  ~  -  _  -  ~  -  _
 * leaving sys-kernel/linux-headers in /var/db/repos/crossdev
 * leaving sys-libs/glibc in /var/db/repos/crossdev
 * leaving sys-devel/binutils in /var/db/repos/crossdev
 * leaving sys-devel/gcc in /var/db/repos/crossdev
 * leaving dev-debug/gdb in /var/db/repos/crossdev
 * leaving metadata/layout.conf alone in /var/db/repos/crossdev
  _  -  ~  -  _  -  ~  -  _  -  ~  -  _  -  ~  -  _  -  ~  -  _  -  ~  -  _  -  ~  -  _
 * Log: /var/log/portage/cross-x86_64-generic-linux-gnu-binutils.log
 * Emerging cross-binutils ...                                                                                                                           [ ok ]
 * Log: /var/log/portage/cross-x86_64-generic-linux-gnu-gcc-stage1.log
 * Emerging cross-gcc-stage1 ...                                                                                                                         [ ok ]
 * Log: /var/log/portage/cross-x86_64-generic-linux-gnu-linux-headers.log
 * Emerging cross-linux-headers ...                                                                                                                      [ ok ]
 * Log: /var/log/portage/cross-x86_64-generic-linux-gnu-glibc.log
 * Emerging cross-glibc ...

 * error: glibc failed :(
 * 
 * If you file a bug, please attach the following logfiles:
 * /var/log/portage/cross-x86_64-generic-linux-gnu-info.log
 * /var/log/portage/cross-x86_64-generic-linux-gnu-glibc.log.xz
 * /var/tmp/portage/cross-x86_64-generic-linux-gnu/glibc*/temp/glibc-config.logs.tar.xz
```

Well, that was interesting.

Examining `/var/log/portage/cross-x86_64-generic-linux-gnu-glibc.log`, we see a couple problems.
```
Calculating dependencies  
 * IMPORTANT: 15 news items need reading for repository 'gentoo'.
 * Use eselect news read to view new items.

Unable to unshare: EPERM (for FEATURES="ipc-sandbox network-sandbox")
...
... done!
Dependency resolution took 1.89 s (backtrack: 0/20).


!!! All ebuilds that could satisfy "cross-x86_64-generic-linux-gnu/glibc" have been masked.
!!! One of the following masked packages is required to complete your request:
- cross-x86_64-generic-linux-gnu/glibc-9999::crossdev (masked by: package.mask, missing keyword)
- cross-x86_64-generic-linux-gnu/glibc-2.39-r5::crossdev (masked by: missing keyword)
- cross-x86_64-generic-linux-gnu/glibc-2.39-r4::crossdev (masked by: missing keyword)
- cross-x86_64-generic-linux-gnu/glibc-2.38-r13::crossdev (masked by: missing keyword)
- cross-x86_64-generic-linux-gnu/glibc-2.37-r10::crossdev (masked by: missing keyword)
- cross-x86_64-generic-linux-gnu/glibc-2.36-r8::crossdev (masked by: missing keyword)
- cross-x86_64-generic-linux-gnu/glibc-2.35-r11::crossdev (masked by: missing keyword)
- cross-x86_64-generic-linux-gnu/glibc-2.34-r14::crossdev (masked by: missing keyword)
- cross-x86_64-generic-linux-gnu/glibc-2.33-r14::crossdev (masked by: missing keyword)
- cross-x86_64-generic-linux-gnu/glibc-2.32-r8::crossdev (masked by: missing keyword)
- cross-x86_64-generic-linux-gnu/glibc-2.31-r7::crossdev (masked by: missing keyword)
- cross-x86_64-generic-linux-gnu/glibc-2.19-r3::crossdev (masked by: missing keyword)

For more information, see the MASKED PACKAGES section in the emerge
man page or refer to the Gentoo Handbook.
```

First, the gentoo docker container doesn't have the necessary privileges to allow `emerge` to call [`unshare`](https://linux.die.net/man/1/unshare).  Second, it looks like gentoo doesn't package GNU libc version 2.26.

The permissions issue can be solved by running a privileged docker container, or by prefixing the `emerge` command with `FEATURES="-ipc-sandbox -network-sandbox -pid-sandbox"`.  (The latter method is preferable in for use in `Dockerfile`s.)

The most maintainable way to solve the missing GNU libc version is to [create an overlay repository](https://flewkey.com/blog/2021-03-29-gentoo-overlays-for-noobs.html) with the missing ebuilds.  To create an ebuild file for glibc 2.26, I started with a copy of [the 2.39 version](https://gitweb.gentoo.org/repo/gentoo.git/tree/sys-libs/glibc/glibc-2.39-r5.ebuild), and modified it to build version 2.26.  This included generating a patch set from [gentoo's fork of the GNU libc repository](https://gitweb.gentoo.org/fork/glibc.git/refs/).

After adding the overlay repository with `eselect repository add rescued-ebuilds git https://github.com/mprzybylski/rescued-ebuilds.git && emerge --sync rescued-ebuilds`, I tried again with...
```shell
FEATURES="-ipc-sandbox -network-sandbox -pid-sandbox" \
        crossdev --target x86_64-generic-linux-gnu --gcc '~14.1.0' \
        --libc '~2.26' --kernel '~4.14' --ex-gdb --portage --verbose
```
Eventually, the glibc build got stuck with `make` burning 100% CPU with no further forward progress.

`top`
```text
Tasks:  12 total,   3 running,   9 sleeping,   0 stopped,   0 zombie
%Cpu(s): 16.4 us,  6.2 sy,  0.0 ni, 77.4 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :  64026.5 total,   1847.7 free,  40709.8 used,  21469.0 buff/cache
MiB Swap: 131072.0 total, 127837.9 free,   3234.1 used.  11955.5 avail Mem 

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
 131046 portage   20   0   17564   5376   1408 R  99.7   0.0   0:36.07 make 
  97764 root      20   0   75952  69104   7552 R  50.7   0.1   0:46.75 emerge
      1 root      20   0    4344   2048   1536 S   0.0   0.0   0:00.05 bash
  97443 root      20   0    6064   3200   2176 S   0.0   0.0   0:00.04 crossdev
 100272 portage   20   0    2488   1280   1152 S   0.0   0.0   0:00.00 sandbox  
 100273 portage   20   0    9384   7168   2304 S   0.0   0.0   0:00.02 bash
 100289 portage   20   0   10740   6648   1664 S   0.0   0.0   0:00.05 bash
 100550 portage   20   0    5800   3328   2176 S   0.0   0.0   0:00.00 bash
 100552 portage   20   0    3856   2048   1408 S   0.0   0.0   0:00.00 make
 100553 portage   20   0   18276   3968   1408 S   0.0   0.0   0:00.16 make
 122478 root      20   0    4344   2688   2176 S   0.0   0.0   0:00.02 bash
 123250 root      20   0    3584   1536   1280 R   0.0   0.0   0:00.04 top 
```

After spending more time than I would have liked searching the web for an explanation,
I discovered that the version of gentoo my development container is based on ships with make 4.4
which is [incompatible with older versions of glibc](https://github.com/crosstool-ng/crosstool-ng/issues/1946).
Worse yet, the gentoo repo doesn't have any versions of make older than 4.4:
```text
19fbddfa0474 /var/db/repos/gentoo/dev-build/make # ls
Manifest  files  make-4.4.1-r1.ebuild  make-9999.ebuild  metadata.xml
```

Fortunately, the gentoo repository did have earlier versions of `make` [at some point](https://gitweb.gentoo.org/repo/gentoo.git/commit/dev-build/make?id=7ef71dd2f0e15a353a958b2572de2f2c353c6afb).  So I copied `make-4.3-r1.ebuild` and its related patches to [rescued-ebuilds](https://github.com/mprzybylski/rescued-ebuilds/tree/main/dev-build/make) and added `<dev-build/make-4.4` to the `BDEPEND` variable in [glibc-2.26-r7.ebuild](https://github.com/mprzybylski/rescued-ebuilds/blob/main/sys-libs/glibc/glibc-2.26-r7.ebuild), ran `emerge --sync rescued-ebuilds` in my development container, and tried again.

This time, the toolchain was created successfully.

# Using in a CMake project

To use this toolchain in a CMake project,
create a [toolchain file](https://cmake.org/cmake/help/latest/manual/cmake-toolchains.7.html), i.e.:

`/cmake/project_root/toolchain.cmake`:
```
set(TOOLCHAIN_PREFIX x86_64-generic-linux-gnu)

set(CMAKE_C_COMPILER ${TOOLCHAIN_PREFIX}-gcc)
set(CMAKE_CXX_COMPILER ${TOOLCHAIN_PREFIX}-g++)
```

...and add `set(CMAKE_TOOLCHAIN_FILE ${CMAKE_CURRENT_LIST_DIR}/toolchain.cmake)` to the top-level `CMakeLists.txt` _before_ the first `project()` command.

This will automatically build your project against the correct GNU libc and Linux kernel headers and produce binaries that are compatible with any Linux distribution with glibc 2.26 or newer.

Additionally, if you happen to be using an IDE like [CLion](https://www.jetbrains.com/clion/), CLion will automagically pick up the correct compilers and header files.

# BPF tooling

In addition to a C/C++ toolchain tailored to the majority of your users' environments,
you will also need tooling for generating and manipulating eBPF object files.
The LLVM project's Clang compiler is the most mature compiler for generating eBPF objects from C source files.
I also strongly recommend [bpftool](https://github.com/libbpf/bpftool) for post-processing and testing eBPF object files
generated by Clang.

# Example

A dockerized implementation of this environment can be found at
[https://github.com/mprzybylski/bpf-iotrace/blob/main/.devcontainer/Dockerfile](https://github.com/mprzybylski/bpf-iotrace/blob/main/.devcontainer/Dockerfile)

# Up next
[Working with dev containers in CLion]({% post_url 2024-05-10-Working-with-dev-containers-in-CLion %})

# Footnotes
[^1]: Apologies to [Green Day](https://youtu.be/Soa3gO7tL-c)
[^2]: "доверяй, но проверяй", _("Trust, but verify.")_: Any project built this way still needs to be tested thoroughly against any Linux distribution version it claims to support.
