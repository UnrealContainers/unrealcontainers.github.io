---
layout: post
title: Identifying application runtime dependencies
author: Adam Rehn
updated: 2021-07-30
tagline: A toolkit for identifying the runtime libraries and associated data that applications require in order to run correctly inside containers.
---

## Overview
{:.no_toc}

The first step required in order to port any existing application to a container-based environment is to identify the dependencies that need to be encapsulated alongside it in a container image. This task is typically straightforward for applications that you have developed yourself since all of the relevant information is already known (and ideally documented), but identifying the dependencies for large, complex or proprietary third-party software can represent a significant undertaking that involves a great deal of investigation. In addition, the container base images for any given operating system will often be optimised for small filesizes and will therefore omit components that are present by default in standard installations of that same operating system. This can lead to oversights in software documentation whereby some dependencies are not listed because they are always present on developer workstations and are therefore assumed to unconditionally ship with the operating system itself.

This blog post presents a set of tools and accompanying strategies for identifying application runtime dependencies under different operating systems. This toolkit has been developed and refined throughout the process of porting a variety of complex open source and proprietary software to containers, and significantly lowers the barrier to entry for developers porting existing applications to both Linux and Windows containers. **This blog post will continue to be updated over time as the toolkit is further refined based on additional experience.**


## Contents
{:.no_toc}

* TOC
{:toc}


## Types of dependencies

Broadly speaking, there are three main types of dependencies that an application may require which will need to be identified:

- **Runtime libraries loaded at application startup:** these are [shared libraries](https://en.wikipedia.org/wiki/Library_(computing)#Shared_libraries) that an application was linked against when it was compiled, as well as any dependencies of those shared libraries. If any of these runtime libraries are absent then the application will fail to start.

- **Runtime libraries loaded during application execution:** these are shared libraries that the application's code loads programmatically at runtime using the [LoadLibrary](https://docs.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-loadlibraryw) function under Windows or the [dlopen](https://pubs.opengroup.org/onlinepubs/9699919799/functions/dlopen.html) function under POSIX based operating systems, as well as any dependencies of those shared libraries. These runtime libraries are not required in order for the application to start, but an error will be produced if the application attempts to load them and they are absent. Individual applications can decide how to handle such an error, but common responses include continuing to run with reduced functionality or immediately halting execution.

- **Configuration data:** these are pieces of data that the application or its shared library dependencies load programmatically at runtime to configure the application's behaviour. Configuration data can take many different forms, commonly including files on the filesystem, environment variables, or [registry entries](https://docs.microsoft.com/en-us/windows/win32/sysinfo/registry) under Windows. Individual applications will react differently to the absence of configuration data, with common responses including falling back to predetermined default values, continuing to run with reduced functionality or immediately halting execution.


## Approaches for identifying dependencies

### Analysis on the host system

The most comfortable approach for many developers when attempting to identify an application's dependencies is to perform analysis directly on the host operating system. Once dependencies have been identified then the target container environment can be checked for their presence or absence. This has the benefit of allowing the use of familiar tools and workflows, including graphical tools under Windows. The downside to this approach is that comparing the host environment to the container environment can be an extremely tedious and labour intensive process, and small details related to system configuration data are often difficult to identify as relevant when inspecting large swathes of irrelevant information. As a result, it is often necessary to perform subsequent analysis inside the container environment after completing analysis in the host environment.

This approach was utilised heavily during the initial development of the [ue4-docker](../../../docs/obtaining-images/ue4-docker) project in 2018, particularly under Windows where most available analysis tools were graphical in nature and thus could not be used inside a container environment. The tedious and labour intensive nature of the approach led to a great deal of frustration, and motivated the subsequent development of tools to better facilitate performing analysis directly inside the container environment. **This approach is useful as a starting point only, and performing analysis inside a container is recommended instead.**

### Analysis inside a container

The most effective approach to identify required application dependencies for use in a container is to perform analysis directly in the container environment itself. This immediately highlights the absence of any required runtime libraries or configuration data as the operating system or the application emits error messages, or the application fails to function. Rather than requiring a thorough inventory of dependencies and a detailed comparison between multiple environments, the identification workflow becomes largely a process of elimination as errors are inspected and addressed. (Comparisons to the host environment can of course be performed as needed when diagnosing individual errors, but analysis performed on the host system in this workflow is directed at inspecting specific details and thus avoids the downsides of the approach described in the previous section.) The self-documenting nature of Dockerfiles also ensures that dependencies are recorded as a matter of necessity as they are identified.

The limitation of this approach is that it relies on the availability of appropriate analysis tools which function correctly inside a container environment. This is typically not a concern when working with Linux containers due to their ability to run almost all workloads that function on a host system, including graphical tools. For Windows containers however, this precludes the use of most popular analysis tools due to their graphical nature and the inability of Windows containers to display GUI elements. As such, the ongoing development of sophisticated command-line analysis tools for Windows is critical to improving the developer experience when porting existing applications to Windows containers.


## Identifying dependencies under Linux

### Identifying runtime libraries

#### Identifying libraries loaded at application startup

The first port of call when identifying the runtime libraries that a Linux application depends upon is the venerable [ldd](https://man7.org/linux/man-pages/man1/ldd.1.html) tool. This tool recursively identifies the runtime libraries that an application loads at startup, and prints the locations of these libraries if they can be found. The output of this tool looks like so:

{% highlight bash %}
$ ldd ./example
    linux-vdso.so.1 (0x00007fff50da0000)
    mylibrary.so => not found
    libstdc++.so.6 => /lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007f3fbb330000)
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f3fbb13e000)
    libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f3fbafef000)
    /lib64/ld-linux-x86-64.so.2 (0x00007f3fbb530000)
    libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f3fbafd4000)
{% endhighlight %}

Dependencies that were located successfully (e.g. system libraries such as `libc` and `libstdc++`) are printed alongside their absolute filesystem path and a hexadecimal memory address indicating the offset at which the library was loaded. Dependencies that could not be found (in this case `mylibrary`) are marked as such, allowing developers to easily identify missing runtime libraries.

When a missing library is identified, the next step is typically to determine which package can be installed through the distribution's package manager to provide the library. The specific method used to determine the appropriate package depends on the particular Linux distribution, but under Debian-based distributions such as Ubuntu and its derivatives you can use the [apt-file](https://wiki.debian.org/apt-file) tool for this purpose. Once identified, the package can be included in a container image by adding an installation command to its Dockerfile.

There are a couple of important things to note when using the `ldd` tool:

- The recursive search can only find the dependencies of libraries that were located successfully. As such, you should invoke the tool in an iterative manner as you locate each missing dependency, which will in turn reveal the dependencies of that dependency and provide a more complete picture of the set of runtime libraries required in order to run the application.

- There may sometimes be instances where `ldd` is unable to locate a runtime library but the application itself loads it successfully when it is run. This typically happens when the application is loaded via a wrapper script that modifies environment variables such as [LD_LIBRARY_PATH](https://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html#AEN80) that influence the manner in which the system searches for shared libraries. In these cases you will need to ensure that you run `ldd` in a shell whose environment variables match those that will be present when the application runs in order to ensure accurate results.

#### Identifying libraries loaded at runtime

Once all of the application's startup dependencies are present and the application is able to run then the next step is to identify any runtime libraries that the application loads programmatically during the course of execution. The diagnostic functionality required for this task is already built into the operating system's dynamic loader infrastructure, so no additional tools are necessary. Simply execute the application with the environment variable [LD_DEBUG](https://man7.org/linux/man-pages/man8/ld.so.8.html#ENVIRONMENT) set to the value `libs` and log output will be printed that looks similar to what is shown below (***comments added for clarity***):

{% highlight bash %}
$ LD_DEBUG=libs ./dlopen-example
    
    # Load the `libdl` system library, which the application is linked against
    find library=libdl.so.2 [0]; searching
     search cache=/etc/ld.so.cache
      trying file=/lib/x86_64-linux-gnu/libdl.so.2
    
    # Load the `libstdc++` system library, which the application is linked against
    find library=libstdc++.so.6 [0]; searching
     search cache=/etc/ld.so.cache
      trying file=/lib/x86_64-linux-gnu/libstdc++.so.6
    
    # Load the `libc` system library, which the application is linked against
    find library=libc.so.6 [0]; searching
     search cache=/etc/ld.so.cache
      trying file=/lib/x86_64-linux-gnu/libc.so.6
    
    # Load the `libm` system library, which the application is linked against
    find library=libm.so.6 [0]; searching
     search cache=/etc/ld.so.cache
      trying file=/lib/x86_64-linux-gnu/libm.so.6
    
    # Load the `libgcc_s` system library, which the application is linked against
    find library=libgcc_s.so.1 [0]; searching
     search cache=/etc/ld.so.cache
      trying file=/lib/x86_64-linux-gnu/libgcc_s.so.1
    
    # Initialise the libraries the application is linked against and start the application
    calling init: /lib/x86_64-linux-gnu/libc.so.6
    calling init: /lib/x86_64-linux-gnu/libgcc_s.so.1
    calling init: /lib/x86_64-linux-gnu/libm.so.6
    calling init: /lib/x86_64-linux-gnu/libstdc++.so.6
    calling init: /lib/x86_64-linux-gnu/libdl.so.2
    initialize program: ./dlopen-example
    transferring control: ./dlopen-example
    
    # The application calls `dlopen("libpthread.so")`, which is found successfully
    find library=libpthread.so [0]; searching
     search cache=/etc/ld.so.cache
      trying file=/lib/x86_64-linux-gnu/libpthread.so
    calling init: /lib/x86_64-linux-gnu/libpthread.so
    
    # The application calls `dlopen("my-dynamic-lib.so")`, which is searched for but cannot be found
    find library=my-dynamic-lib.so [0]; searching
     search cache=/etc/ld.so.cache
     search path=/lib/x86_64-linux-gnu/tls/x86_64/x86_64:/lib/x86_64-linux-gnu/tls/x86_64:/lib/x86_64-linux-gnu/tls/x86_64:/lib/x86_64-linux-gnu/tls:/lib/x86_64-linux-gnu/x86_64/x86_64:/lib/x86_64-linux-gnu/x86_64:/lib/x86_64-linux-gnu/x86_64:/lib/x86_64-linux-gnu:/usr/lib/x86_64-linux-gnu/tls/x86_64/x86_64:/usr/lib/x86_64-linux-gnu/tls/x86_64:/usr/lib/x86_64-linux-gnu/tls/x86_64:/usr/lib/x86_64-linux-gnu/tls:/usr/lib/x86_64-linux-gnu/x86_64/x86_64:/usr/lib/x86_64-linux-gnu/x86_64:/usr/lib/x86_64-linux-gnu/x86_64:/usr/lib/x86_64-linux-gnu:/lib/tls/x86_64/x86_64:/lib/tls/x86_64:/lib/tls/x86_64:/lib/tls:/lib/x86_64/x86_64:/lib/x86_64:/lib/x86_64:/lib:/usr/lib/tls/x86_64/x86_64:/usr/lib/tls/x86_64:/usr/lib/tls/x86_64:/usr/lib/tls:/usr/lib/x86_64/x86_64:/usr/lib/x86_64:/usr/lib/x86_64:/usr/lib		(system search path)
      trying file=/lib/x86_64-linux-gnu/tls/x86_64/x86_64/my-dynamic-lib.so
      trying file=/lib/x86_64-linux-gnu/tls/x86_64/my-dynamic-lib.so
      trying file=/lib/x86_64-linux-gnu/tls/x86_64/my-dynamic-lib.so
      trying file=/lib/x86_64-linux-gnu/tls/my-dynamic-lib.so
      trying file=/lib/x86_64-linux-gnu/x86_64/x86_64/my-dynamic-lib.so
      trying file=/lib/x86_64-linux-gnu/x86_64/my-dynamic-lib.so
      trying file=/lib/x86_64-linux-gnu/x86_64/my-dynamic-lib.so
      trying file=/lib/x86_64-linux-gnu/my-dynamic-lib.so
      trying file=/usr/lib/x86_64-linux-gnu/tls/x86_64/x86_64/my-dynamic-lib.so
      trying file=/usr/lib/x86_64-linux-gnu/tls/x86_64/my-dynamic-lib.so
      trying file=/usr/lib/x86_64-linux-gnu/tls/x86_64/my-dynamic-lib.so
      trying file=/usr/lib/x86_64-linux-gnu/tls/my-dynamic-lib.so
      trying file=/usr/lib/x86_64-linux-gnu/x86_64/x86_64/my-dynamic-lib.so
      trying file=/usr/lib/x86_64-linux-gnu/x86_64/my-dynamic-lib.so
      trying file=/usr/lib/x86_64-linux-gnu/x86_64/my-dynamic-lib.so
      trying file=/usr/lib/x86_64-linux-gnu/my-dynamic-lib.so
      trying file=/lib/tls/x86_64/x86_64/my-dynamic-lib.so
      trying file=/lib/tls/x86_64/my-dynamic-lib.so
      trying file=/lib/tls/x86_64/my-dynamic-lib.so
      trying file=/lib/tls/my-dynamic-lib.so
      trying file=/lib/x86_64/x86_64/my-dynamic-lib.so
      trying file=/lib/x86_64/my-dynamic-lib.so
      trying file=/lib/x86_64/my-dynamic-lib.so
      trying file=/lib/my-dynamic-lib.so
      trying file=/usr/lib/tls/x86_64/x86_64/my-dynamic-lib.so
      trying file=/usr/lib/tls/x86_64/my-dynamic-lib.so
      trying file=/usr/lib/tls/x86_64/my-dynamic-lib.so
      trying file=/usr/lib/tls/my-dynamic-lib.so
      trying file=/usr/lib/x86_64/x86_64/my-dynamic-lib.so
      trying file=/usr/lib/x86_64/my-dynamic-lib.so
      trying file=/usr/lib/x86_64/my-dynamic-lib.so
      trying file=/usr/lib/my-dynamic-lib.so
    
    # Perform cleanup now that application execution has completed
    calling fini: ./dlopen-example [0]
    calling fini: /lib/x86_64-linux-gnu/libdl.so.2 [0]
    calling fini: /lib/x86_64-linux-gnu/libstdc++.so.6 [0]
    calling fini: /lib/x86_64-linux-gnu/libm.so.6 [0]
    calling fini: /lib/x86_64-linux-gnu/libgcc_s.so.1 [0]
    calling fini: /lib/x86_64-linux-gnu/libpthread.so [0]
{% endhighlight %}

As can be seen in the example output above, runtime libraries that are loaded successfully will be represented by search lines followed by an output line containing `calling init:` with the absolute filesystem path for the library file. Libraries which are searched for but cannot be found lack this additional output line and typically feature a larger number of search lines as the loader checks, and eventually exhausts, all candidate paths for the library file.

Once any missing runtime libraries are identified, the process for determining how to include them in a container image is the same as that described in the previous section. As before, each dependency which is identified may in turn have its own dependencies, so an iterative workflow is necessary in order to ensure all dependencies are identified and satisfied.

One important nuance to be aware of when identifying dependencies which are loaded programmatically during application execution is that different code paths may load different runtime libraries. It may be necessary to test an application with a variety of different command-line flags in order to trigger distinct code paths for different modes or behaviours that require different dependencies.

### Identifying configuration data

#### Identifying data access through the filesystem

Configuration data can take many different forms, but most common types of data are accessed through the filesystem. This is particularly true under Linux, where files are used as an abstraction mechanism to expose a wide variety of resources to user applications. Whenever a file is accessed through the filesystem, the application makes a [system call](https://man7.org/linux/man-pages/man2/syscalls.2.html) to the Linux kernel in order to request access. The [strace](https://man7.org/linux/man-pages/man1/strace.1.html) tool provides functionality to trace all of the system calls made by an application, and its filtering options make it easy to specifically identify all file open operations, like so (***comments added for clarity***):

{% highlight bash %}
$ strace --trace=open,openat ./config-example

# These file operations are related to loading runtime libraries and can be ignored
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libstdc++.so.6", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libgcc_s.so.1", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libm.so.6", O_RDONLY|O_CLOEXEC) = 3

# This is the application attempting to open `config.txt` in read mode and failing to find the file
openat(AT_FDCWD, "config.txt", O_RDONLY) = -1 ENOENT (No such file or directory)

+++ exited with 0 +++
{% endhighlight %}

All file open operations are listed as [open or openat](https://man7.org/linux/man-pages/man2/open.2.html) entries, which include the filesystem path being accessed, the requested file mode (e.g. read-only or both read and write), along with the result of the operation and an error message if it failed. Failures to access configuration data files are easy to identify, although determining how to include the required files in a container image may require a more involved investigation to determine where the file should be sourced from. If the file is not specific to the application then it might be provided by a system package, in which case tools previously described in the section [Identifying libraries loaded at application startup](#identifying-libraries-loaded-at-application-startup) such as `apt-file` can be used to determine which package provides the file.

#### Identifying environment variable access

Environment variables are a common source of configuration data that are not accessed via the filesystem. Since environment variables are stored in the memory of each running process, accessing them is simply a matter of reading the contents of this memory and does not involve a system call that can be identified using `strace`. Instead, it is necessary to identify calls to the [getenv](https://man7.org/linux/man-pages/man3/getenv.3.html) function. (It is worth noting that applications might also conceivably access the [environ](https://man7.org/linux/man-pages/man7/environ.7.html) array directly, but this is less common and identifying this sort of access is extremely difficult without resorting to the use of a debugger.)

The most popular tool for identifying calls to functions in system libraries under Linux is [ltrace](https://man7.org/linux/man-pages/man1/ltrace.1.html). It is worth noting that some versions of `ltrace` don't work with all applications [due to incompatibilities with certain compiler flags](https://bugzilla.redhat.com/show_bug.cgi?id=1333481), so if you find that the tool doesn't provide any output for a given application then it is also worth trying the alternative [uftrace](https://github.com/namhyung/uftrace) tool. Both tools provide similar tracing functionality and also include filtering options that make it easy to identify calls to `getenv()`.

Identifying calls with `ltrace` looks like so:

{% highlight bash %}
$ ltrace -x getenv ./config-example
getenv@libc.so.6("CONFIG_FILE_PATH") = nil
+++ exited (status 0) +++

{% endhighlight %}

Identifying calls with `uftrace` looks like so:

{% highlight bash %}
$ uftrace live --auto-args --patch=getenv --filter=getenv ./config-example
# DURATION     TID     FUNCTION
  50.370 us [ 31125] | getenv("CONFIG_FILE_PATH") = "NULL";
{% endhighlight %}


## Identifying dependencies under Windows

{% include alerts/base.html
	class="alert-primary"
	icon="fas fa-book"
	title="Recommended reading:"
	content="For a more complete understanding of Windows containers, including their quirks and limitations, see the [Windows Containers](../../../docs/concepts/windows-containers) page."
%}

### Identifying runtime libraries

#### Sourcing missing runtime libraries

Under Linux, including a missing runtime library in a container image is often as simple as determining the appropriate package and adding a command to a Dockerfile to install it through the distribution's package manager. Under Windows however, package management systems such as [Chocolatey](https://chocolatey.org/) and [winget](https://docs.microsoft.com/en-us/windows/package-manager/winget/) focus primarily on installing applications, and typically only feature packages for a small handful of runtime libraries. This makes the process of identifying an appropriate source for a runtime library more difficult, and can represent a frustrating obstacle for new developers porting existing applications to Windows containers.

Use the following strategies as a guide to assist in identifying appropriate sources for libraries:

- **Source system libraries from the Windows client base image.** Microsoft provides [multiple base images for Windows containers](../../../docs/concepts/windows-containers#windows-base-image-variants), with each base image designed to represent a different tradeoff between application compatibility and image size. If you are using a container image derived from the Windows Server Core base image (which is the most popular image for porting existing applications) then you will often encounter missing system libraries that are present in standard installations of Windows. These system libraries are typically included in the [mcr.microsoft.com/windows](https://hub.docker.com/_/microsoft-windows) base image, which represents a Windows client installation and features almost all of the libraries and services from a standard desktop installation. It is of course possible to derive your container image from this base image directly in order to improve compatibility, but the prohibitively large image size makes it poorly suited to production use. For an example of how to use a [Docker multi-stage build](https://docs.docker.com/develop/develop-images/multistage-build/) to copy runtime libraries from the client Windows base image to a Windows Server Core image, see the blog post [Offscreen rendering in Windows containers](../offscreen-rendering-in-windows-containers/).

- **Source the Microsoft Visual C++ runtime libraries from Chocolatey.** Most C++ applications will require the version of the Microsoft Visual C++ runtime libraries that corresponds to the compiler version that they were built with. Although it is possible to explicitly download the installers for these runtime libraries and run them in a Dockerfile, it is far simpler and easier to use Chocolatey to install the [vcredist-all](https://community.chocolatey.org/packages/vcredist-all) package, which automatically downloads and installs all available versions of the Microsoft Visual C++ runtime libraries.

- **Source DirectX runtime libraries from the June 2010 redistributable package.** Although a number of DirectX runtime libraries are included with Windows and can be sourced from the Windows client base image as described above, there are some DirectX libraries that must still be downloaded from Microsoft. Currently the best available source for these runtime libraries is the [June 2010 DirectX End-User Runtimes](https://www.microsoft.com/en-au/download/details.aspx?id=8109) installer, which can also be automatically downloaded and installed through the [directx](https://community.chocolatey.org/packages/directx) Chocolatey package.

#### The dll-diagnostics tool

One of the key limitations of Windows containers is that they are [unable to run graphical applications](https://github.com/microsoft/Windows-Containers/issues/27) and support command-line use only. Since most popular application analysis tools for Windows are graphical in nature and feature only limited command-line functionality, this constraint makes the process of debugging dependency issues inside Windows containers far more difficult than their Linux counterparts, where robust command-line analysis tools are readily available and [graphical applications can be used if needed](https://unrealcontainers.com/docs/use-cases/linux-sandboxed-editor). In an effort to help address this situation, I created the [dll-diagnostics](https://github.com/adamrehn/dll-diagnostics) tool, which provides a comprehensive set of commands for inspecting and debugging the manner in which applications locate and load runtime libraries. The tool exposes a single command-line interface named `dlldiag` (in a similar vein to Microsoft's [dxdiag](https://support.microsoft.com/en-us/windows/open-and-run-dxdiag-exe-dad7792c-2ad5-f6cd-5a37-bf92228dfd85)) which supports a variety of subcommands for common analysis tasks, and includes [ready-made container images](https://hub.docker.com/r/adamrehn/dll-diagnostics) to act as a starting point for developers who are porting existing applications to Windows containers.

#### Identifying libraries loaded at application startup

There are two `dlldiag` subcommands available for identifying the libraries that an application loads at startup and determining whether any of them are missing. The simpler of these subcommands is `dlldiag deps`, which identifies the direct dependencies of an application and tests whether each library can be loaded, like so:

{% highlight plain %}
> dlldiag deps example.exe

DLL Diagnostic Tools version 0.0.18
Copyright (c) 2019-2021 Adam Rehn

Parsing module header and identifying direct dependencies... done.

Parsed module details:
Module:          C:\Users\Adam\Desktop\example.exe
Type:            Executable
Architecture:    x64

Attempting to load the module's direct dependencies:

api-ms-win-crt-heap-l1-1-0.dll       Loaded successfully
api-ms-win-crt-locale-l1-1-0.dll     Loaded successfully
api-ms-win-crt-math-l1-1-0.dll       Loaded successfully
api-ms-win-crt-runtime-l1-1-0.dll    Loaded successfully
api-ms-win-crt-stdio-l1-1-0.dll      Loaded successfully
KERNEL32.dll                         Loaded successfully
MSVCP140.dll                         Loaded successfully
my-library.dll                       Error 126: The specified module could not be found.
VCRUNTIME140.dll                     Loaded successfully

Important note regarding errors:

Errors loading indirect dependencies are propagated by LoadLibrary(), which
means any errors displayed above that indicate missing or corrupt modules may
in fact be referring to a child dependency of a direct dependency, rather than
the direct dependency itself.

Use the `dlldiag trace` command to inspect loading errors for a module in detail.
{% endhighlight %}

As can be seen above, any runtime libraries that cannot be loaded will be listed alongside the relevant error details. It is worth noting that although this subcommand is somewhat limited due to the fact that it does not identify dependencies recursively, it also doesn't require administrative privileges or rely on any external tools and is therefore much easier to run on the host system in the event that a comparison needs to be made between the host environment and the container environment.

The more comprehensive subcommand available for identifying libraries that an application loads at startup is `dlldiag trace`, which uses the Windows debugger to trace the load operations for all of an application's direct dependencies, allowing it to identify the dependencies of those libraries in a recursive manner. This subcommand requires that the Windows debugger be present in the environment in which it is run, and also requires administrative privileges. These requirements may be somewhat cumbersome on a host system but aren't an issue inside a container environment, since the container images for dll-diagnostics include the debugger already and Windows containers run as an administrator user by default. The output of this subcommand looks similar what is shown below:

{% highlight plain %}
> dlldiag trace example.exe

DLL Diagnostic Tools version 0.0.18
Copyright (c) 2019-2021 Adam Rehn

Parsing module header and identifying non delay-loaded dependencies... done.

Identifying the module's delay-loaded dependencies... done.

Parsed module details:
Module:          C:\Users\Adam\Desktop\example.exe
Type:            Executable
Architecture:    x64

The module imports 9 direct dependencies:
api-ms-win-crt-heap-l1-1-0.dll
api-ms-win-crt-locale-l1-1-0.dll
api-ms-win-crt-math-l1-1-0.dll
api-ms-win-crt-runtime-l1-1-0.dll
api-ms-win-crt-stdio-l1-1-0.dll
KERNEL32.dll
MSVCP140.dll
my-library.dll
VCRUNTIME140.dll

Performing LoadLibrary() trace for C:\Users\Adam\Desktop\example.exe...
Performing LoadLibrary() trace for api-ms-win-crt-heap-l1-1-0.dll...
Performing LoadLibrary() trace for api-ms-win-crt-locale-l1-1-0.dll...
Performing LoadLibrary() trace for api-ms-win-crt-math-l1-1-0.dll...
Performing LoadLibrary() trace for api-ms-win-crt-runtime-l1-1-0.dll...
Performing LoadLibrary() trace for api-ms-win-crt-stdio-l1-1-0.dll...
Performing LoadLibrary() trace for KERNEL32.dll...
Performing LoadLibrary() trace for MSVCP140.dll...
Performing LoadLibrary() trace for my-library.dll...
Performing LoadLibrary() trace for VCRUNTIME140.dll...
Done.

Summary of LdrLoadDll calls:
api-ms-win-crt-heap-l1-1-0.dll       Loaded successfully
api-ms-win-crt-locale-l1-1-0.dll     Loaded successfully
api-ms-win-crt-math-l1-1-0.dll       Loaded successfully
api-ms-win-crt-runtime-l1-1-0.dll    Loaded successfully
api-ms-win-crt-stdio-l1-1-0.dll      Loaded successfully
C:\Users\Adam\Desktop\example.exe    Loaded successfully
KERNEL32.dll                         Loaded successfully
MSVCP140.dll                         Loaded successfully
my-library.dll                       Error 126: The specified module could not be found.
VCRUNTIME140.dll                     Loaded successfully

Summary of LdrpLoadDllInternal calls:
C:\Users\Adam\Desktop\example.exe    Loaded successfully
C:\WINDOWS\SYSTEM32\ucrtbase.dll     Loaded successfully
KERNEL32.dll                         Loaded successfully
MSVCP140.dll                         Loaded successfully
my-library.dll                       Error 126: The specified module could not be found.
VCRUNTIME140.dll                     Loaded successfully

Summary of LdrpMinimalMapModule calls:
C:\Users\Adam\Desktop\example.exe    Mapped successfully

Summary of LdrpResolveDllName calls:
example.exe       C:\Users\Adam\Desktop\example.exe
my-library.dll    Error 126: The specified module could not be found.
{% endhighlight %}

The output is grouped by each internal Windows API call found in the trace output from the debugger. As before, libraries that failed to load are listed next to the relevant error message to aid in diagnosing the exact cause of the load failure. It is worth noting that a failure to load a dependency of a given library will also result in a failure to load the library itself, so it is important to manually check which libraries are indeed present in order to distinguish between errors which indicate the absence of a dependency and those that simply indicate the absence of one of the libraries that the dependency itself depends upon.

The recursive search performed by `dlldiag trace` can only find the dependencies of libraries that were located successfully. As such, you should invoke the subcommand in an iterative manner as you locate each missing dependency, which will in turn reveal the dependencies of that dependency and provide a more complete picture of the set of runtime libraries required in order to run the application.

#### Identifying libraries loaded at runtime

Once all of the application's startup dependencies have been satisfied and the application is able to run then the next step is to use the `dlldiag graph` subcommand to identify any runtime libraries that the application loads programmatically during the course of execution. This powerful subcommand uses the Microsoft Research [Detours](https://github.com/microsoft/Detours) library (discussed in the section below) to instrument the Windows API calls used to load libraries and collects detailed data to reconstruct the full graph of dependencies between libraries. Unfortunately this instrumentation approach only works when the application is able to run, so it cannot be used to identify startup dependencies like the subcommands discussed in the previous section.

The `dlldiag graph` subcommand features a number of options to configure its behaviour, but the simplest invocation looks like so:

{% highlight plain %}
> dlldiag graph loadlibrary-example.exe

DLL Diagnostic Tools version 0.0.18
Copyright (c) 2019-2021 Adam Rehn

Parsing module header and detecting architecture... done.

Parsed module details:
Module:          C:\Users\Adam\Desktop\loadlibrary-example.exe
Type:            Executable
Architecture:    x64

Running executable C:\Users\Adam\Desktop\loadlibrary-example.exe with arguments [] and instrumenting all LoadLibrary() calls...

C:\WINDOWS\SYSTEM32\VCRUNTIME140.dll:
    LoadLibraryExW "api-ms-win-core-synch-l1-2-0" -> C:\WINDOWS\System32\KERNELBASE.dll
    LoadLibraryExW "api-ms-win-core-fibers-l1-1-1" -> C:\WINDOWS\System32\KERNELBASE.dll

C:\WINDOWS\System32\KERNELBASE.dll:
    This module did not load any libraries.

C:\Users\Adam\Desktop\loadlibrary-example.exe:
    LoadLibraryA "my-dynamic-lib.dll" -> The specified module could not be found.

C:\WINDOWS\System32\ucrtbase.dll:
    LdrLoadDll "api-ms-win-core-synch-l1-2-0" -> C:\WINDOWS\System32\KERNELBASE.dll
    LdrLoadDll "api-ms-win-core-fibers-l1-1-1" -> C:\WINDOWS\System32\KERNELBASE.dll
    LdrLoadDll "api-ms-win-core-localization-l1-2-1" -> C:\WINDOWS\System32\KERNELBASE.dll
    LdrLoadDll "api-ms-win-appmodel-runtime-l1-1-2" -> C:\WINDOWS\SYSTEM32\kernel.appcore.dll

C:\WINDOWS\SYSTEM32\kernel.appcore.dll:
    This module did not load any libraries.
{% endhighlight %}

The output is grouped by each module (executable or library) detected in the instrumentation data and lists each of the libraries that the module attempted to load. Libraries that were loaded successfully are listed alongside the absolute filesystem path to the resolved library file, whilst libraries that failed to load are listed alongside the relevant error message. As always, each missing dependency that is identified and resolved may in turn have its own dependencies and you should invoke the subcommand in an iterative manner to identify the full set of required runtime libraries. It is also worth exploring the available options for the subcommand, including the `--output` flag to print application output after the instrumentation summary and the `--extended` flag to display more detailed instrumentation data and highlight special load operations such as when the [DirectX runtime loads a User Mode Driver (UMD)](../enabling-vendor-specific-graphics-apis-in-windows-containers/#wddm-architecture).

As mentioned in the Linux counterpart to this section, one factor to be aware of when identifying dependencies which are loaded programmatically during application execution is that different code paths may load different runtime libraries. It may be necessary to test an application with a variety of different command-line flags in order to trigger distinct code paths for different modes or behaviours that require different dependencies.

### Identifying configuration data

#### The Detours library and associated tools

Unlike Linux, Windows does not ship with comprehensive command-line tools to inspect the mechanisms by which applications might access configuration data. Fortunately, the [Detours](https://github.com/microsoft/Detours) project by Microsoft Research provides a powerful library and accompanying tools for instrumenting Windows API calls and collecting data about application behaviour. In addition to the core library and tools, the project also provides an extensive [set of samples](https://github.com/microsoft/Detours/wiki/Samples) that demonstrate instrumenting various subsets of the Windows API. These tools and samples are ideal for capturing information about the numerous ways that application configuration data can be accessed.

Pre-built binaries are not provided for the Detours project, so developers need to [compile the code from source](https://github.com/microsoft/Detours/wiki/FAQ#compiling-with-detours-code) before making use of the tools and samples. For simplicity, it is recommended that you perform the build on the host system and then use the built tools inside your container environment. The build scripts provided by the project only build the code for the architecture specified by the selected [developer tools command prompt](https://docs.microsoft.com/en-us/cpp/build/building-on-the-command-line?view=msvc-160#developer_command_prompt_shortcuts), so if you want to build both 32-bit and 64-bit binaries then you will need to perform the build twice using the appropriate command prompt (i.e. "x86 Native Tools Command Prompt" for 32-bit and "x64 Native Tools Command Prompt" for 64-bit.) Once the code is built, the binaries for the tools and samples can be found under the `bin.X86` subdirectory for the 32-bit version and `bin.X64` for the 64-bit version, and the appropriate files can be copied into a container image. **Be sure to use the matching binaries for the architecture of the application you are analysing!**

#### Identifying Windows registry access

The [Tracereg](https://github.com/microsoft/Detours/wiki/SampleTracereg) Detours sample instruments the Windows API functions related to registry access. Like most of the samples, it is designed to be injected into an application's process via the [Withdll](https://github.com/microsoft/Detours/wiki/SampleWithdll) tool and to send log output to the [Syelog](https://github.com/microsoft/Detours/wiki/SampleSyelog) tool. As such you will need to open two separate command prompts inside the container (one for the instrumented application and one for the log output), which can be achieved by running the [docker exec](https://docs.docker.com/engine/reference/commandline/exec/) command in two separate command prompts on the host system.

In the first command prompt, run the Syelog executable `syelogd.exe`, which will wait for connections from other Detours instrumentation samples and print any log output that it receives.

In the second command prompt, run the application with instrumentation enabled like so:

{% highlight plain %}
> withdll.exe /d:trcreg64.dll C:\Users\Adam\Desktop\config-example.exe

withdll.exe: Starting: `C:\Users\Adam\Desktop\config-example.exe'
withdll.exe:   with `C:\Users\Adam\Desktop\Detours\bin.X64\trcreg64.dll'
{% endhighlight %}

The log output displayed by Syelog will include a number of extraneous entries for unrelated Windows API calls, so search for entries pertaining to [registry functions](https://docs.microsoft.com/en-us/windows/win32/sysinfo/registry-functions) (most of which are prefixed with `Reg`):

{% highlight bash %}
# This log line corresponds to this function call:
# RegOpenKeyExW(HKEY_LOCAL_MACHINE, L"my-config-key", 0, KEY_ALL_ACCESS, nullptr)
trcreg64: 001 RegOpenKeyExW(ffffffff80000002,my-config-key,0,f003f,0) -> 57
{% endhighlight %}

#### Identifying filesystem access and environment variable access

Although the [Tracebld](https://github.com/microsoft/Detours/wiki/SampleTracebld) Detours sample instruments the Windows API functions related to both filesystem access and environment variable access, it produces very different output to the other instrumentation samples and appears to be designed only for a handful of specific use cases. As such, relying on this sample is not recommended. Instead, the general-purpose [Traceapi](https://github.com/microsoft/Detours/wiki/SampleTraceapi) sample should be used, which instruments over a thousand different Windows API calls. Since there is no filtering functionality available, the log output will most likely need to be searched using other tools in order to identify the relevant entries, such as the Find feature in [Windows Terminal](https://github.com/microsoft/terminal), or the search functionality in a text editor or IDE.

Just like the Tracereg sample discussed in the section above, the Traceapi sample is designed to be injected into an application's process via the Withdll tool and to send log output to the Syelog tool. As above, this necessitates opening two separate command prompts inside the container using `docker exec` or a similar mechanism.

In the first command prompt, run the Syelog executable `syelogd.exe`, which will wait for connections from other Detours instrumentation samples and print any log output that it receives.

In the second command prompt, run the application with instrumentation enabled like so:

{% highlight plain %}
> withdll.exe /d:trcapi64.dll C:\Users\Adam\Desktop\config-example.exe

withdll.exe: Starting: `C:\Users\Adam\Desktop\config-example.exe'
withdll.exe:   with `C:\Users\Adam\Desktop\Detours\bin.X64\trcapi64.dll'
{% endhighlight %}

Search the lengthy log output displayed by Syelog for entries pertaining to [filesystem functions](https://docs.microsoft.com/en-us/windows/win32/fileio/file-management-functions) and [environment variable functions](https://docs.microsoft.com/en-us/windows/win32/procthread/environment-variables):

{% highlight bash %}
# This log line corresponds to this function call:
# GetEnvironmentVariableW(L"CONFIG_FILE_PATH", buf, sizeof(buf))
trcapi64: 001 +GetEnvironmentVariableW(CONFIG_FILE_PATH,2770affc00,200)

# This log line corresponds to the application attempting to open the file "config.txt"
trcapi64: 001 +CreateFileW(config.txt,80000000,3,ce204ff660,2ae00000003,80,0)
{% endhighlight %}
