---
layout: post
title: "A simple pair of eBPF tracepoint handlers"
category: libbpf-based tracing from scratch
tags: 
  - Linux
  - software
  - engineering
  - BPF
  - eBPF
  - development
  - bpftool
  - tracepoints
date: 2024-06-17 09:00 -0800
---
!["Hello, World" in green on a black background](/images/hello-world.png)
# Previously
[eBPF programs the CMake way]({% post_url 2024-05-17-eBPF-programs-the-CMake-way %})

# Starting small

Eventually, `bpf-iotrace` is going to instrument every system call that can do filesystem I/O.
However, for starters, I wanted to write about the eBPF equivalent of "Hello, world,"
since the structure of an eBPF program is so wildly different from most other types of software.

eBPF program entry points have a great deal in common with JavaScript events like `onclick`,
`onmouseover`, or `onload`.
But instead of web browser events, they are kernel events like entering or returning
from a system call, a process exit, or network packet being received.
eBPF programs can also be attached to kernel function entry or return exit points, (`kprobe`s or `kretprobes`),
or user space function entry and exit points, (`uprobe`s or `uretprobe`s).
Each eBPF "program" is a function that handles one of those kernel events.
The eBPF programs described in this post will be attached to the `sys_enter_read` and `sys_exit_read` tracepoints in
the Linux kernel.

Those handlers will work together to print the name of the program that called `read()`,
its PID, the file descriptor number, the `count` argument to `read()` and `read()`'s return value.

## eBPF source file boilerplate

### Includes

#### `<bpf/bpf_helpers.h>`
This libbpf header file provides definitions for eBPF helper functions and macros for commonly used eBPF attributes.

#### `<linux/bpf.h>`
This kernel header file defines additional bpf-related kernel constants.

#### `<stdint.h>`
This C standard library header file provides terse definitions for signed and unsigned integers of varying widths.

### License section
Most BPF helper functions for retrieving interesting data from the kernel 
or pushing it to userspace can only be used in GPL-licensed programs.
`bpf-iotrace` is going to rely on the `probe_kernel_read()`, `bpf_perf_event_output()`, `bpf_get_current_task()`,
`bpf_ktime_get_ns()` helper functions and all of them will only work with GPL-licensed eBPF programs.
In fact, the Linux kernel will refuse to load any program that uses a GPL-only helper but is not GPL-licensed.

Developers can use the `SEC()` macro from `bpf_helpers.h` to specify the license for a BPF object file in a way that
can be understood by the Linux kernel:

```
char LICENSE[] SEC("license") = "GPL";
```

The line above declares a `char` array called `LICENSE` containing the value `GPL`, and tells the compiler to put it
in an ELF section called `license` in the compiled object file.
See [the Linux source code for `license_is_gpl_compatible()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/license.h)
for other GPL-compatible values for `LICENSE`.

To identify other GPL-only eBPF helper functions, search the kernel source tree for `.gpl_only	= true`.
(Please note there is a "hard" tab character between `.gpl_only` and `=`.)

# Context is everything
Every eBPF program receives a pointer to a struct containing contextual data specific to the type of eBPF program
being invoked.
For kernel tracepoints, you can determine the layout of this context struct by examining the `format` file for
the tracepoint in `/sys/kernel/debug/tracing/events`,
i.e. `/sys/kernel/debug/tracing/events/syscalls/sys_enter_read/format` or 
`/sys/kernel/debug/tracing/events/syscalls/sys_exit_read/format`.

The `field:` descriptions in...
```text
root@thinky-winks:/sys/kernel/debug/tracing/events/syscalls/sys_enter_read# cat format
name: sys_enter_read
ID: 755
format:
        field:unsigned short common_type;       offset:0;       size:2; signed:0;
        field:unsigned char common_flags;       offset:2;       size:1; signed:0;
        field:unsigned char common_preempt_count;       offset:3;       size:1; signed:0;
        field:int common_pid;   offset:4;       size:4; signed:1;

        field:int __syscall_nr; offset:8;       size:4; signed:1;
        field:unsigned int fd;  offset:16;      size:8; signed:0;
        field:char * buf;       offset:24;      size:8; signed:0;
        field:size_t count;     offset:32;      size:8; signed:0;

print fmt: "fd: 0x%08lx, buf: 0x%08lx, count: 0x%08lx", ((unsigned long)(REC->fd)), ((unsigned long)(REC->buf)), ((unsigned long)(REC->count))
```
...can be translated to:
```
struct sys_enter_read_tp_ctx {
    uint8_t inaccessible[16];
    uint32_t fd;
    char * buf;
    uint64_t count;
};
```

A similar `format` file exists for the `sys_exit_read` tracepoint, and it can be translated to a
struct in a similar manner.

Please note that the first 16 bytes of these context structs are inaccessible to an eBPF program.
Attempting to access the fields in that region will cause the verifier will reject the program with error messages like
`invalid bpf_context access off=4 size=4`.

# Correlating entry and exit events
One of the interesting challenges that arises in eBPF programming is how to capture all the data specified by
requirements when individual eBPF event handlers only have partial views of that data via their context arguments.
In this example, we can get the file descriptor number and the count argument from the context passed to an entry
handler for `read()`.
However, the return value is only available from the exit handler.
How does a programmer pass data from the entry handler of a system call to the exit handler?
This is one of the problems that eBPF maps were designed to solve.

## eBPF hash maps
The Linux kernel provides [several different types of eBPF maps](https://docs.kernel.org/bpf/maps.html) including a
hash map capable of handling primitive or composite, (i.e. `struct`), key and value types.

Only one system call can be running on a given thread at a time.  This makes the current thread ID a useful key for
storing and retrieving data captured from a system call's entry trace point handler.

The `bpf_get_current_pid_tgid()` BPF helper function returns a 64-bit, unsigned integer containing the current task's
thread group ID and PID, specifically `current_task->tgid << 32 | current_task->pid`.
`tgid` is what most Linux users think of as the "process ID."  The `pid` field is the thread ID.

We can define a struct for storing `read()`s `fd` and `count` arguments...
```
struct sys_read_state {
    uint64_t count;
    unsigned int fd;
};
```
...and we can define a BPF hash map for storing and retrieving that state:
```
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 1024);
    __type(key, uint32_t);
    __type(value, struct sys_read_state);
} sys_read_state_map SEC(".maps");
```

The `__uint()` and `__type()` macros are defined in `bpf_helpers.h`.
The struct above is written to the `.maps` ELF section in the compiled eBPF object file.
`libbpf` will read its data and create a map named sys_read_state_map with 32-bit signed integer keys,
`struct sys_read_state` values, and a capacity of 8192 entries.

It is important to note that eBPF maps are created in kernel memory, which is a finite resource, and it is important
to size them for their expected usage.  If a map is too small, eBPF programs may not be able to capture data
for events they are supposed to be tracking.  If a map is too large, it will either waste system memory, or libbpf
will be unable to create it.
For `bpf-iotrace` that means estimating the number of instrumented threads that could be making I/O system calls
simultaneously.  For this example, we will pick a low but still generous number for `max_entries`.

An eBPF map entry can be read with the `bpf_map_lookup_elem()`
[helper function](https://man7.org/linux/man-pages/man7/bpf-helpers.7.html).
An entry can be inserted or replaced with the `bpf_map_update_elem()` helper function.

It is also important to delete map entries that have outlived their usefulness, such as a `sys_read_state_map` entry
where the underlying `read()` call has returned.
This can be done with `bpf_map_delete_elem()`.

# Other macros and helpers

## `SEC()`
The `SEC()` macro specifies the ELF section for a particular variable or function.  The `license` and `.maps` ELF
sections have already been described above.
The other major use of the `SEC()` macro is assigning eBPF programs to their own, named ELF sections.

`libbpf` expects program section names to follow certain conventions:
* The prefix of the section name must be a valid identifier for the program type, i.e. `tp` or `tracepoint` for
  tracepoints, `kprobe` and `kretprobe` for kernel function entry and return event handlers, etc.
  (See `section_defs[]` in [libbpf.c](https://github.com/libbpf/libbpf/blob/master/src/libbpf.c) for a complete list
  of valid section name prefixes.)
* If the program is meant to attach directly to a tracepoint, its suffix should match its path under
  `/sys/kernel/debug/tracing/events/`, i.e. `SEC("tp/syscalls/sys_enter_read")` or `SEC("tp/syscalls/sys_exit_read")`.
  This allows programs in an eBPF object file to be automatically attached by `bpftool prog loadall ... autoattach`.
  If the program is meant to be [tail-called](https://ebpf-docs.dylanreimerink.nl/linux/concepts/tail-calls/) instead,
  the developer is free to choose their own naming convention.

## ` long bpf_get_current_comm(void *buf, u32 size_of_buf)`
This helper copies the `comm` attribute of the current task into the specified buffer.

## `bpf_trace_printk(const char *fmt, u32 fmt_size, ...)`
This helper allows eBPF programs to write formatted strings to user space.
It takes a pointer to a buffer containing a format string, a size argument for the format buffer, and up to three
additional variables.  `bpf_trace_printk()` supports the following conversion specifiers.
`%d, %i, %u, %x, %ld, %li, %lu, %lx, %lld, %lli, %llu,  %llx,  %p,  %s`.
However, it does not support any sort of modifier like conversion width, zero-padding, places after the decimal point,
etc.

It is also important to note that eBPF programs are limited to a 512-byte stack.  So variables containing format
strings and calls to `bpf_trace_printk()` should be enclosed in their own curly braces so that they go out of scope
and are kicked off the stack as soon as possible.

It is also worth noting that `bpf_trace_printk()` is slow, and initializing a format string eats into eBPF's
already-limited instruction budget.  As such, it should only be used sparingly and temporarily for debugging.
As noted in the [`bpf-helpers(7)` man page](https://man7.org/linux/man-pages/man7/bpf-helpers.7.html),
developers looking to ship large amounts of data from eBPF programs to user space should use 
[perf buffers](https://ebpf-docs.dylanreimerink.nl/linux/map-type/BPF_MAP_TYPE_PERF_EVENT_ARRAY/) or the
[eBPF ring buffer](https://www.kernel.org/doc/html/next/bpf/ringbuf.html) instead.

# Trying it all out

After adding both tracepoint handlers and supporting data structures to
[read.bpf.c](https://github.com/mprzybylski/bpf-iotrace/blob/2de3cf6da463c8b25047180f86530d433cb7f569/src/bpf/read.bpf.c)
and compiling it, we can load it with [`bpftool`](https://github.com/libbpf/bpftool)...
```
sudo bpftool prog loadall read.bpf.c.o /sys/fs/bpf/read autoattach
```
...and examine its output with `sudo cat /sys/kernel/debug/tracing/trace_pipe`:
```
...
     firefox-bin-29640   [003] ...21 419084.735437: bpf_trace_printk: Captured read() syscall: comm: firefox-bin, pid: 29640, fd: 35, 
     firefox-bin-29640   [003] ...21 419084.735437: bpf_trace_printk: count: 1, ret: 1

 TerminalEmulato-789099  [004] ...21 419084.736266: bpf_trace_printk: Captured read() syscall: comm: TerminalEmulato, pid: 563462, fd: 352, 
 TerminalEmulato-789099  [004] ...21 419084.736267: bpf_trace_printk: count: 8192, ret: 4095

 ProtocolFromCha-674460  [005] ...21 419084.737123: bpf_trace_printk: Captured read() syscall: comm: ProtocolFromCha, pid: 674328, fd: 73, 
 ProtocolFromCha-674460  [005] ...21 419084.737124: bpf_trace_printk: count: 17408, ret: 403

            Xorg-28027   [007] ...21 419084.738881: bpf_trace_printk: Captured read() syscall: comm: Xorg, pid: 28027, fd: 16, 
            Xorg-28027   [007] ...21 419084.738882: bpf_trace_printk: count: 1024, ret: 32

     gnome-shell-28203   [006] ...21 419084.739110: bpf_trace_printk: Captured read() syscall: comm: gnome-shell, pid: 28203, fd: 3, 
     gnome-shell-28203   [006] ...21 419084.739110: bpf_trace_printk: count: 16, ret: 8

 TerminalEmulato-789099  [004] ...21 419084.739349: bpf_trace_printk: Captured read() syscall: comm: TerminalEmulato, pid: 563462, fd: 352, 
 TerminalEmulato-789099  [004] ...21 419084.739350: bpf_trace_printk: count: 8192, ret: 4095

 Isolated Web Co-30503   [006] ...21 419084.740530: bpf_trace_printk: Captured read() syscall: comm: Isolated Web Co, pid: 30503, fd: 22, 
 Isolated Web Co-30503   [006] ...21 419084.740530: bpf_trace_printk: count: 1, ret: 1
 ...
```

To detach these eBPF programs and remove them from the kernel, run `sudo rm -rf /sys/fs/bpf/read`.

As you can see in the output above, these eBPF programs are doing what I wanted them to do,
but they are _noisy_ because they treat every system call exactly the same way.

To minimize `bpf-iotrace`'s impact we will want our eBPF trace point handlers to extract complete data for
filesystem I/O calls from processes of interest while returning early when invoked by other processes or by
socket I/O operations.

# Up next
Filtering events in eBPF programs

# References
* [Linux kernel BPF documentation table of contents](https://docs.kernel.org/bpf/index.html)
  * [BPF standardization](https://docs.kernel.org/bpf/standardization/index.html)
    * [BPF ABI](https://docs.kernel.org/bpf/standardization/abi.html)
  * [BPF licensing](https://docs.kernel.org/bpf/bpf_licensing.html)
  * [BPF hash maps](https://docs.kernel.org/bpf/map_hash.html)
* [BTF-style map definitions for eBPF](https://lwn.net/ml/netdev/20190531202132.379386-7-andriin@fb.com/)
* [`bpf-helpers(7)` man page](https://man7.org/linux/man-pages/man7/bpf-helpers.7.html)
* [eBPF perf buffers](https://ebpf-docs.dylanreimerink.nl/linux/map-type/BPF_MAP_TYPE_PERF_EVENT_ARRAY/)

(Where do libbpf-bootstrap programs get their context structures?!?  
For the most part, they don't.  Only a few tracepoints have ctx structs defined in vmlinux.h)