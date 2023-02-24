---
layout: post
title: "bpf-iotrace: Defining Requirements"
category: libbpf-based tracing from scratch
tags: Linux software engineering BPF eBPF development performance syscall libbpf filesystems benchmarks fio
date: 2023-02-12 00:00:00 -0800
---
# Previously
[A New Tool for an Old Problem]({% post_url 2023-02-11-A-New-Tool-For-an-Old-Problem %})

# Getting organized
Any software engineering task more complex than a trivial bug fix needs a checklist.  This is especially true of new projects and new features.

The checklist can take any form that works most efficiently for a developer, their team, or their organization.  JIRA issues, ClickUp tasks, or a text or Markdown file on your desktop are all viable options[^1], but writing a good checklist has an even more fundamental pre-requisite: good requirements.

Requirements describe a project's desired end result.  Good requirements are a set of concise and unambiguous statements that focus on _what_ a project or product must do.  Good requirements _shouldn't_ prescribe _how_ a developer or team should implement its requirements.  Those decisions should be left to the experts on the implementing team as much as possible.[^2]

You may be fortunate enough to work with a product or project manager or engineering lead who understands their problem space really well, who can just _hand_ you a nice, cleanly-written, ready-to-implement list of requirements.  If not, don't fret.

If you were given any written requirements at all, read them carefully, and do your best to get written clarification of any ambiguities or inconsistencies you find.  Once you feel like you fully understand those requirements, you can break them down into a task list in the best way that works for you or your team.

If you were given a complex software project without any requirements, kick off your design process by writing requirements for yourself based on your understanding of the problem.  Then check in with your stakeholders to see if you are on the right track.  Once you are, you can confidently translate them to a task list as you move on to design and implementation.  This approach also works when you are working solo and developing your own ideas.

I like to structure my requirements as much as I can like an [IETF RFC](https://www.ietf.org/standards/rfcs/).  Thinking about what functionality a project must deliver in terms of [RFC 2119 keywords](https://www.ietf.org/rfc/rfc2119.txt) has really helped me separate the "whats" of the deliverable from the "hows" of the implementation.

The working draft of `bpf-iotrace`'s requirements may be found [here](https://github.com/mprzybylski/bpf-iotrace/blob/main/REQUIREMENTS.md), and they are discussed in greater detail below.

# `bpf-iotrace` requirements

## Target operating system
Word-of-mouth suggests that Linux v4.14 is the oldest feasible target for a BPF-based utility.  `bpf-iotrace` will also be linked against libc6 v2.27.  So any Linux host with that version of libc6 or newer should be compatible. 

## Implementation languages
The languages to be used on a software project are one of the very few "hows" that it makes sense to specify as a requirement since there is moderate coupling between a programming language and its target environment(s).  There may also be personnel, time, or other resource constraints that dictate the selection of a particular programming language.  In the case of `bpf-iotrace`, one of its main objectives is to serve as a demonstration project for libbpf-based application and modern C++ development techniques.  So `bpf-iotrace` will be written in C++, C, and eBPF assembly.

## Output
`bpf-iotrace` should be able to take an application that is essentially a black box and see where and how it is reading and writing data.  Its output must allow administrators and dev-ops engineers to create filesystems with optimal configurations for hosting that application.  `bpf-iotrace` should also provide the insights necessary to back those filesystems with the most cost-effective storage technologies.  Its output must also inform benchmarks that can rapidly qualify those file systems.

Given that most applications and especially database servers and NoSQL data stores often interact with large directory trees, `bpf-iotrace`'s output must be formatted to allow analysis of I/O data for a top-level directory, an individual file, and everything in between.[^3]

This means `bpf-iotrace` must save its metrics on a per-file basis, but containers add an extra wrinkle: how does `bpf-iotrace` reliably identify files in different containers when the file has the same name, (i.e. `ib_logfile0` in two separate MySQL Sever containers on the same Docker host or Kube node)?  This means `bpf-iotrace` must use the mount namespace ID, (i.e. `readlink /proc/<pid>/ns/mnt`), _and_ an absolute path to uniquely identify a file.

### Per-file  metrics

* Since `bpf-iotrace` will be used for troubleshooting, it shall record an error count for each I/O system call.
* A FIO benchmark can be configured to generate a distribution of I/O sizes in a variety of system calls, so `bpf-iotrace` shall record a histogram of size arguments and return values, (usually the number of bytes read or written), for each I/O system call.
* `bpf-iotrace` shall record a histogram of sequence lengths when it identifies runs of sequential read or write calls.
* `bpf-iotrace` shall also record the following data on a per-file basis:
  * A histogram of the number of bytes written between `fsync()` calls
  * Total bytes read
  * The entry time for the first read system call
  * The return time for the last successful read system call
  * Total bytes written
  * The entry time for the first write system call
  * The return time for the last successful write system call
  * Total bytes written
  * Total bytes written before the first `fsync()` system call if `fsync()` has only been called once on the file descriptor, or between the two most recent `fsync()` calls, if `fsync()` has been called more than once.
  * The return time of the most recent `fsync()` system call
  * The entry time of the first write system call
  * The return time of the last successful write system call
  * The number of times a write was issued to the same file offset as a previous write
  * A histogram of `fsync()` latencies

The above metrics should allow a user to characterize an application's I/O including calculating read and write bandwidths for a single file or an entire directory tree.

## Instrumentation
`bpf-iotrace` shall include BPF instrumentation for the following system calls:
* `close()`
* `fsync()`
* `sendfile()`
* `pread64()`
* `preadv()`
* `preadv2()`
* `read()`
* `readv()`
* `pwrite64()`
* `pwritev()`
* `pwritev2()`
* `write()`
* `writev()`

`bpf-iotrace` shall also include trace point and/or kprobe instrumentation necessary to track asynchronous I/O operations.

The list above is intended to capture every I/O function an application might use to read from or write to a filesystem.  If you think it is missing anything, please [let me know here](https://github.com/mprzybylski/bpf-iotrace/discussions/1).

## Filtering
Another interesting feature of BPF programs is that they will execute _any and every_ time the trace point or kernel function they were attached to is encountered.  This can be incredibly powerful in terms of visibility, but it can also be incredibly _noisy_.  In the case of `read()`, `readv()`, `write()`, `writev()`, and `sendfile()`, those functions can interact with regular files, or file descriptors for sockets, (network I/O).  At the very least, `bpf-iotrace()` instrumentation must ignore network I/O all the time.  More generally, `bpf-iotrace` must only trace I/O operations on named, regular files.

Typically, a performance or dev-ops engineer is only going to be interested in the behavior of a small subset of commands or process IDs running on a system.  Therefore, `bpf-iotrace` must provide options that allow a user to specify a set of process IDs or commands to be monitored exclusively.

## TUI reporting
I am thinking of adding text-based user interface, (TUI), reporting and analysis to `bpf-iotrace` as a future enhancement.  If you have a wish list for functionality that you would like it to include, please [let me know here](https://github.com/mprzybylski/bpf-iotrace/discussions/1).

## Time series I/O metrics
I am thinking of adding time-series file I/O metrics to `bpf-iotrace` as a future enhancement.  If you have ideas or requests for lightweight ways to record that data, and the specific time series metrics you would like to see included, please [let me know here](https://github.com/mprzybylski/bpf-iotrace/discussions/1).

# Footnotes
[^1]: I'm well aware that these aren't the _only_ options for organizing a complex project, and if you have another favorite, let me know [here](https://github.com/mprzybylski/mprzybylski.github.io/discussions/1)
[^2]: A project's stakeholders and/or its developers _should_ create documentation on how a project will be implemented.  However, those implementation plans should be discussed, refined, and approved in _design reviews_.  The design review process can be as formal or informal as the stake-holders need or want, but it is incredibly helpful for everyone to be on the same page regarding what a project's requirements _are_ and how they will be satisfied.
[^3]: Note that we are _not_ specifying the low-level format for `bpf-iotrace`'s output file(s).  Since there are no external constraints like a customer requirement or a downstream platform consuming `bpf-iotrace`'s data, we can leave it up to the implementors to select a format that supports the analysis requirements mentioned above.  If there were customer or consumer constraints, then it would be perfectly appropriate to specify them as requirements.

# Up next
Creating a repeatable build environment for eBPF development.

# References
* [https://fio.readthedocs.io/en/latest/fio_doc.html](https://fio.readthedocs.io/en/latest/fio_doc.html)
* [https://www.ietf.org/standards/rfcs/](https://www.ietf.org/standards/rfcs/)
* [https://www.ietf.org/rfc/rfc2119.txt](https://www.ietf.org/rfc/rfc2119.txt)