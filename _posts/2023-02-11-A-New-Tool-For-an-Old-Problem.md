---
layout: post
title: "A New Tool for an Old Problem"
category: libbpf-based tracing from scratch
tags: 
  - Linux
  - software
  - engineering
  - BPF
  - eBPF
  - development
  - performance
  - syscall
  - libbpf
  - filesystems
  - benchmarks
  - fio
date: 2023-02-11 09:30:00 -0800
---

# Once upon a time...

Around 2016, my colleagues and I were troubleshooting bizarre performance problems with an application that I will call Monolithic Observability Product for Enterprises, or MOPE.  It was a web application backed by a single MySQL database which used the InnoDB storage engine to store everything.  The database and web app would start up, merrily ingest data, serve up data requested via the UI... and grind to a halt about an hour later.  Restart the database and web server, and the cycle would repeat.

When we inquired about the platform being used to host MOPE, we learned it was a virtual machine running on a popular hypervisor and backed by SAN storage which was, in turn, backed by rotating media.  (Old-school, spinning hard drives.)  The nature of the storage subsystem immediately made us suspicious about _its_ performance. So we shut down MOPE and ran the IOZone storage benchmark prescribed by MOPE's documentation against the VM.

IOZone said the storage was fine.

We eventually discovered what was going on. MOPE is an extremely write-heavy workload for MySQL, ingesting metrics, traces, and events from MOPE's agents.  MySQL's InnoDB storage engine handles writes by updating database pages in memory and writing a sequential log of those changes to disk in one of at least two "redo" log files.  These redo log files are pre-allocated on disk, and InnoDB writes to them in round-robin fashion. Eventually, InnoDB gets around to flushing those dirty database pages in memory back to disk so that transactions in the redo log file can be retired.  Since sequential disk I/O is always faster than random I/O, and multiple changes to the same database page can be coalesced in memory, this scheme allows InnoDB to perform fast _**and**_ durable writes.

But what happens when one of those redo log files fills up?  InnoDB does two things simultaneously:
1. It switches redo log writes over to the next log file in the round-robin.
2. It flushes dirty database pages referenced by that full log file back to disk using asynchronous writes.

This log switch happened pretty reliably after MOPE had been running for about an hour.

The dirty page flushes represented a large volume of random I/O.

The InnoDB redo logs and the `.ibd` files containing its database pages were on the same filesystem.

All of this means greatly increased contention for the same set of disk arms on the SAN.  This greatly increased the latency of writes to InnoDB's log files which, in turn, backed all the way up into MOPE's web application.  Agent data fell on the floor when it couldn't be ingested fast enough.  MOPE's UI became unusable as read queries started taking hundreds of times longer than they should have.

We rescued this instance of MOPE by migrating the InnoDB redo logs and data files to separate virtual disks, making sure to put the redo logs on SSD-backed storage.

But there was a bigger problem: MOPE's documented IOZone storage benchmark was _wrong_.  This was going to burn us over and over again unless we understood _why_ it was wrong and corrected it.

# Truing up the filesystem benchmark

From the InnoDB engine source code, I was able to figure out what simulated page flushes should look like: random, 16KiB, asynchronous writes, with each write followed by an `fsync()`.

Figuring out what simulated log writes should look like was a bit more work.  Thanks to Brendan Gregg's excellent [`perf` Examples](https://www.brendangregg.com/perf.html) page, we were able to collect a histogram of InnoDB log write sizes from another busy instance of MOPE.  With that data, I was able to feed some educated guesses to [`fio`](https://fio.readthedocs.io/en/latest/fio_doc.html) and come up with a benchmark that did a much better job at telling us when a MOPE user had provisioned storage that didn't meet requirements.

While the new FIO benchmark made things way better, `perf` could only tell us so much about MySQL's I/O patterns.  Its worst blind spot was that it could tell us virtually nothing about the performance of the asynchronous writes.  It was also difficult to correlate what `perf` could tell us with FIO's benchmark results.  While I was looking for visibility into these blind spots, I started running across articles from Brendan Gregg on an interesting new technology: eBPF.

# eBPF

The original BPF, (BSD Packet Filter or Berkeley Packet Filter, now referred to by some as "classic BPF"), was a way of programming a kernel at runtime with arbitrary packet matching and filtering logic.  It consisted of a virtual machine running within the kernel with its own bytecode instruction set and system calls for loading a BPF program and verifying that it would not crash or lock up the kernel.

_Extended_ BPF or eBPF showed up in the Linux kernel starting in version 3.18.  The extensions in question widened the BPF virtual machine data path from 32 bits to 64 bits, and replaced classic BPF's single accumulator and index register with ten 64-bit general-purpose registers.  It also added kernel helper functions that a developer could call from an eBPF program.  These extensions and helpers greatly expanded the possibilities for the kinds of programs that could be written for eBPF.

It is important to note that every eBPF program is essentially an event handler.  Javascript developers who have worked on handlers for browser events will be quite familiar with this concept.  For everyone else, think of a custom program that runs every time a button is clicked, or when the mouse pointer rolls over an image.  By Linux kernel version 4.7, it was possible to attach a BPF program to the entry or return event for any kernel function that appears in `/proc/kallsyms`, (this is called a kprobe), or any kernel trace point listed in `/sys/kernel/debug/tracing/events`.  Kprobes and trace points are given pointers to structures containing useful data like the arguments passed into an instrumented function, or that function's return value.

With these enhancements, there are no more blind spots MySQL's behavior.  All a developer has to do is write and attach eBPF programs for the right syscall trace points and kernel functions, and that developer can get a picture of MySQL's I/O performance that correlates exactly with the performance numbers that a FIO benchmark generates.  Furthermore, a performance engineer can use the data gleaned from those eBPF programs to tune FIO's benchmarking settings to look more like the real MySQL workload for their application.

A generalized utility based on this approach could also characterize other database servers or NoSQL data stores to improve the accuracy of pre-installation filesystem benchmarks or inform the tuning of a data store's underlying filesystem, (i.e. [ZFS `recordsize`](https://www.percona.com/blog/mysql-zfs-performance-update/)).  This utility would also come in handy for me at home where I could use its output to tune the ZFS datasets underlying the network shares on my home Linux server.

# Interlude

Playing with eBPF to shed more light on these questions went on my nerd bucket list while I was fighting other, higher-priority fires and stayed there for several years.  In 2021, I was approached by the founders of a seed-stage startup who wanted to use eBPF to capture networked API traffic.  I accepted the job and spent almost the next two years teaching myself eBPF, Linux kernel internals, and modern C++.  I built an instrumentation product that captured HTTP API requests and responses in bare metal and containerized environments without modifying application code or configurations.  Along the way, I also encountered and solved fascinating problems related to eBPF development, and accumulated a long list of blog post ideas related to those challenges and the techniques I used to overcome them.  

Since then, I've been laid off twice.

So now, I have time on my hands and a need to market myself for my next paying gig.  Thank goodness for that list of blog post ideas.  Now I just need some code to provide context and examples for those posts...

# `bpf-iotrace`

As I reflected on my journey with eBPF, it occurred to me that the original questions that brought eBPF to my attention--and the tooling idea that resulted--would be the perfect showcase for everything I have learned about eBPF and modern C++ development so far.

As I started looking for a good name for the project, I discovered that `ioprof`, `ioperf`, and [`iotrace`](https://github.com/nicolasgross/iotrace) were already taken.  That wasn't necessarily a problem, since I could differentiate my project by prepending `bpf-` to one of those existing names.  When I looked at the capabilities and functionality, I discovered that [`iotrace`](https://github.com/nicolasgross/iotrace) was the closest in functionality to what I wanted to build.  When I took a closer look at it to make sure I wasn't reinventing the wheel, I noted that it used [`ptrace`](https://linux.die.net/man/2/ptrace) to intercept system calls.  Wasn't there a Brendan Gregg article about `ptrace`?  [Yes.  Yes there was.](https://www.brendangregg.com/blog/2014-05-11/strace-wow-much-syscall.html).

Given the extreme overhead inherent in `ptrace`, `iotrace` looks very much like a wheel _worth_ reinventing.  So I am gradually working on a new open source project called `bpf-iotrace` that will be based on eBPF and [libbpf](https://github.com/libbpf/libbpf).  This blog will host a series of articles describing the process of building that tool from scratch.

# Up next:
[defining requirements for `bpf-iotrace`]({% post_url 2023-02-12-bpf-iotrace__Defining_Requirements %}).

# Additional References
* [https://android.googlesource.com/platform/external/bcc/+/master/docs/kernel-versions.md](https://android.googlesource.com/platform/external/bcc/+/master/docs/kernel-versions.md)
* [https://en.wikipedia.org/wiki/Linux_kernel_version_history](https://en.wikipedia.org/wiki/Linux_kernel_version_history)
* [https://en.wikipedia.org/wiki/Berkeley_Packet_Filter](https://en.wikipedia.org/wiki/Berkeley_Packet_Filter)
