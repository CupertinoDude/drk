# DRK: DynamoRIO as a Linux Kernel Module

Peter Feiner November 17, 2013

## INTRODUCTION

DRK is DynamoRIO as a loadable Linux Kernel module. When DRK is loaded, all
kernel-mode execution (system calls, interrupt & exception handlers, kernel
threads, etc.) happens under the purview of DynamoRIO whereas user-mode execution is
untouched - the inverse of normal DynamoRIO, which instruments a user-mode
process and doesn't touch kernel-mode execution.

## PREREQUISITES

Although in principle DRK should work with any Linux kernel, in practice you're
best off using Ubuntu's 2.6.32-33-generic, which shipped with Ubuntu 10.04. The
coupling arises from DRK's use of a few non-exported kernel interfaces to look
at symbols and get the kernel's memory layout. With some hacking of
kernel_linux/kernel_interface.c, you should be able to get DRK to work with
other kernel versions.

You'll also need Python and SCons.

## BUILDING

Unlike DynamoRIO, DRK doesn't use cmake for configuration and building.
Instead, it has a simple makefile (core/drk.mk) and a static configuration
header (core/configure.h). To configure DRK, you'll have to edit configure.h
and drk.mk.

To build, run these commands:

```
    cd core/
    make -f drk.mk
```

## CLIENTS

Instrumentation clients are built as separate kernel modules. Several example
clients are included in core/kernel_linux/clients. See
core/kernel_linux/modules/Makefile for how the clients are included in the
build.

## RUNNING

The core/drk.py script loads the DRK and client kernel modules and starts
instrumentation. The script reads DynamoRIO options, including the
specification of clients to load, from the dr_options file. For example, to
start DRK with the instruction counting client, instrcount:

```
    cd core/
    echo "-code_api -client_lib instrcount;1;" > dr_options
    sudo ./drk.py --run-locally
```

When the last command returns, you're running from the code cache!

To stop instrumentation, run 
   
``` 
    cd core/kernel_linux
    sudo ./controller exit
```

When sudo returns, you're no longer running from the code cache.

## STATS

You can dump DynamoRIO's stats and kstats as follows:

```
    cd core/kernel_linux
    sudo ./controller stats
    sudo ./controller kstats
```

## TODO

There's a bunch of housekeeping that could be done on the DRK code:

- Use CMake to generate drk.mk and configure.h. Get rid of SCons.

- Move clients from core/kernel_linux/clients to clients/.

- Make drk.py work from any directory.

- Move drk.py to tools/.

- Licensing is a bit of a mess. All of the code should be BSD.

There are some less trivial cleanup tasks that could be done:
    
- Merge dynamorio, dynamorio_controller, and dr_kernel_utils into a single
      kernel module; frankly, I can't remember why I separated it into three
      separate modules in the first place.

- Replace stats & kstats ioctls with sysfs files that can just be read.

- dr_init does not show up in /proc/kallsyms, so clients have to be
      modified to use drinit instead. It would be nice to use drinit so we
      don't have to modify existing clients, but this isn't important because
      it's such a trivial change.

There's a bunch of DynamoRIO stuff that just doesn't work. Making these things
work in DRK will take quite a bit of engineering effort. Making some of these
things work _well_ will probably take some research effort :-) Here's a
list of some major things that don't work:
    
- Running out of space in the code cache. DRK will crash if it tries to
      reset the cache entirely when it runs out of space. Use these options to
      trigger the crash:
        -loglevel 0 -logmask 0x20 -bb_ibl_targets -disable_traces -kstats
        -code_api -no_finite_bb_cache -vm_size 6mb -reset_at_vmm_free_limit 1mb
        -report_reset_vmm_threshold 1000 -skip_out_of_vm_reserve_curiosity

- vmareas is basically a big nop in DRK. Hence DRK has no sort of code
      cache consistency in face of virtual address changes or native code
      changes. Fortunately, Linux doesn't change its code at runtime under
      normal circumstances :-) However, when it does (e.g., ftrace, kprobes,
      CPU hot plugging), you're going to have a bad time.

- 32-bit kernels.

- Shared code caches. DRK uses CPU-private caches to make interrupt
      handling simpler. See the ASPLOS paper for details.

- Traces. As far as I can remember, they work, but they don't help with
      performance at all.

- Changes to the kernel's GS segment. DRK uses Linux's CPU-private data for
      its own CPU-private data. At every entry into kernel-mode, the first
      code cache instruction is SWAP_GS; any native SWAP_GS instructions are
      suppressed. If the kernel were to change its GS usage at all, then DRK
      would break. One solution would be to emulate the kernel's use of GS,
      like userspace DynamoRIO does since the private loader patches.

There are probably a bunch of other DynamoRIO features & code paths that don't
work at all.

## FURTHER READING

My master's thesis explains the why & how of DRK:

> Peter Feiner, Angela Demke Brown, and Ashvin Goel.
> "Comprehensive Kernel Instrumentation via Dynamic Binary Translation."
> International Conference on Architectural Support for Programming Languages
>     and Operating Systems (ASPLOS-12), March 2012, London, UK.
> http://www.cs.toronto.edu/~peter/feiner_asplos_2012.pdf
