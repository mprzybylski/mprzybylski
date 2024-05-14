---
layout: post
title: "Working with dev containers in CLion"
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
- Docker
- "Development Containers"
- JetBrains
- CLion
- Automation
date: 2024-05-10 09:00 -0800
---
![Container hanging from crane](/images/container-on-crane.png)
# Previously
[Creating a build environment for libbpf-based programs]({% post_url 2024-05-09-Creating-a-build-environment-for-libbpf-based-programs %})

# Let my people code

One of my pet peeves as a software engineer is a project that requires a litany of manual steps before I can produce a
working binary, or make full use of CLion's powerful code indexing and navigation features.
So one of the things I try really hard to do when structuring a project's build system is
to minimize the number of manual steps required between cloning its source files and coding, building, and testing.

Docker has always been wonderfully helpful in this regard.
It provides a language for creating repeatable, tailored environments for building software,
and excellent utilities for interacting with those environments.
[CLion's support for](https://www.jetbrains.com/help/clion/connect-to-devcontainer.html)
[Development Containers](https://containers.dev/) makes Docker containers even more powerful
by using them as the basis for easily provisioned, repeatable, _development_ environments.

# The `bpf-iotrace` development container
The most fundamental component of the `bpf-iotrace` development container is
[its Dockerfile](https://github.com/mprzybylski/bpf-iotrace/blob/main/.devcontainer/Dockerfile),
which automates the creation of development environment described in my
[previous post](({% post_url 2024-05-09-Creating-a-build-environment-for-libbpf-based-programs %})).
Next,
there is [`devcontainer.json`](https://github.com/mprzybylski/bpf-iotrace/blob/main/.devcontainer/devcontainer.json)
which references the Dockerfile, and defines additional runtime options for the container including `--privileged`,
and the [`sysfs`](https://www.kernel.org/doc/html/latest/filesystems/sysfs.html) mounts
needed to interact with the kernel's eBPF subsystem.
Those options allow a developer to run gentoo `emerge` commands with all of their sandbox features enabled,
and to run eBPF code directly from the container.
`Dockerfile` and `devcontainer.json` also define an unprivileged user in the container
to minimize file ownership issues with the project source files on the host. 
Finally,
there is [`toolchain.cmake`](https://github.com/mprzybylski/bpf-iotrace/blob/main/.devcontainer/toolchain.cmake)
which tells CMake the names of compilers in container's custom toolchain.

# CLion and development containers in action

CLion can work with development containers in two modes: cloning a project directly into a container,
or mounting a local project in a container.
I prefer the latter because it makes it easier to throw away a container after a failed experiment without losing work.

So far,
the only way I've found to launch a development container with local sources mounted in it from a JetBrains product is
to open the project's `devcontainer.json` in a local instance of CLion...

![devcontainer.json open in local CLion](/images/Working-with-dev-containers-in-CLion/project_with_dot_devcontainer_open.png)

...and then click the ![create dev container](/images/Working-with-dev-containers-in-CLion/create_dev_container_button.png)
button in the editor's left-hand gutter.

![create dev container menu](/images/Working-with-dev-containers-in-CLion/project_with_create_dev_container_menu.png)

I then select "Create Dev Container and Mount Sources..."

This builds the development container.
The process can take a significant amount of time since it has to build GCC,
Clang, and other associated tooling and libraries from source.
(I have plans to speed up this process by building most of the development container in GitHub Actions and hosting
its image in GitHub's Container Registry.)

![dev container build complete](/images/Working-with-dev-containers-in-CLion/building-dev-container-complete.png)

Once the build is complete, I make sure CLion is selected in the drop-down menu at the top of the window, and click the
"Continue" button.
This launches the CLion remote development backend in the development container and connects to it with an
instance of JetBrains Client running on my laptop.
Note the bpf-iotrace source code mounted in my container from my home directory.

![project in dev container](/images/Working-with-dev-containers-in-CLion/project_in_dev_container.png)

# Another development container use case: software packaging for non-native Linux distributions
My daily driver for this work is an x86_64 laptop running Debian 12.
This made it challenging to work on gentoo packages
because part of the workflow involves running commands
like `ebuild ./foo-1.2.3-r4.ebuild manifest` to update checksums in package manifest files.
By defining [a simple development container in my `rescued-ebuilds` overlay](https://github.com/mprzybylski/rescued-ebuilds/tree/main/.devcontainer),
I can easily launch a properly configured environment for working on ebuilds any time I need it.

# Up Next
[Managing dependencies in vcpkg]({% post_url 2024-05-13-Managing-dependencies-in-vcpkg %})

# Additional references

## CMake toolchains
* [CMake toolchains](https://cmake.org/cmake/help/latest/manual/cmake-toolchains.7.html)

## Development Containers
* [Specification](https://containers.dev/implementors/spec/)
* [Other tools that support development containers](https://containers.dev/supporting)

## Development Containers in CLion
* https://www.jetbrains.com/help/clion/prerequisites-for-dev-containers.html