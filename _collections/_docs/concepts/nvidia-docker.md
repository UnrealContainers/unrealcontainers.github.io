---
title: NVIDIA Docker
tagline: What is the NVIDIA Docker runtime and how can it be used to run Linux containers with full GPU acceleration?
quickstart: ["run"]
order: 2
---

{% capture _alert_content %}
- NVIDIA Docker allows containers to access full GPU acceleration.
- OpenGL, OpenCL and CUDA are currently supported. Vulkan support has been promised as a future addition.
- **This only works for Linux containers running on Linux host systems with NVIDIA GPUs.**
{% endcapture %}
{% include alerts/overview.html content=_alert_content %}


## Contents
{:.no_toc}

* TOC
{:toc}


## What is a container runtime?

As explained in [this article discussing container runtimes](https://www.ianlewis.org/en/container-runtimes-part-1-introduction-container-r), the term "container runtime" is used to refer to a variety of software components involved in the process of running containers. These components can be divided into two broad categories:

- **"Low-level" container runtimes:** these runtimes are the software component responsible for actually running containers using the host operating system's native isolation functionality. Low-level container runtimes are typically specific to a particular operating system or family of operating systems.

- **"High-level" container runtimes:** these runtimes are higher-level abstractions that typically sit on top of the low-level runtimes and provide additional convenience features. High-level container runtimes can achieve cross-platform portability by hooking into the appropriate low-level container runtime for any given host platform. A popular example of a cross-platform, high-level container runtime is [containerd](https://containerd.io/).

For the purposes of this discussion we are primarily interested in low-level container runtimes. Here are three examples of popular low-level container runtimes that can be used by Docker:

- [**runc**](https://github.com/opencontainers/runc): this is the default container runtime used by Docker to run containers under Linux host systems. The runc runtime interacts directly with the Linux kernel to run containers.

- [**runhcs**](https://github.com/Microsoft/hcsshim/tree/master/cmd/runhcs): this is the default container runtime used by Docker to run Windows containers under Windows host systems. The runhcs runtime interacts with the [Host Compute Service](https://techcommunity.microsoft.com/t5/Containers/Introducing-the-Host-Compute-Service-HCS/ba-p/382332), which in turn interacts directly with the Windows kernel to run containers. The runhcs runtime is a modified version of the runc runtime.

- [**NVIDIA Docker**](https://github.com/NVIDIA/nvidia-docker): this is the container runtime used to run containers with full GPU acceleration on NVIDIA hardware. The NVIDIA Docker runtime interacts with the GPU drivers on the host system and [uses code from the runc runtime](https://github.com/NVIDIA/nvidia-container-runtime) to interact with the Linux kernel and run containers. The NVIDIA Docker runtime is a modified version of the runc runtime.


## What is the NVIDIA Docker container runtime?

As stated in the final list item of the previous section, [NVIDIA Docker](https://github.com/NVIDIA/nvidia-docker) is a low-level container runtime that is designed to act as a drop-in replacement for the default Linux container runtime [runc](https://github.com/opencontainers/runc). The NVIDIA Docker runtime allows containers to communicate with the GPU drivers on the host system, providing full access to all NVIDIA GPU devices. Containers can then use hardware-accelerated APIs including OpenGL, OpenCL and NVIDIA CUDA. Vulkan support [has been promised as a future addition](https://github.com/NVIDIA/nvidia-docker/wiki/Frequently-Asked-Questions#is-vulkan-supported).

NVIDIA Docker is designed specifically for Linux containers running on Linux host systems. The runtime [does not support Windows containers](https://github.com/NVIDIA/nvidia-docker/wiki/Frequently-Asked-Questions#platform-support), nor can it be used when running Linux containers under Windows or macOS due to the fact that containers are run inside a Linux VM that does not have GPU access. However, Docker clients running under Windows and macOS can still be used to connect to a Docker daemon running under Linux with NVIDIA Docker.


## What steps do I need to follow to make use of NVIDIA Docker?

1. As per the [NVIDIA Docker prerequisites list](https://github.com/NVIDIA/nvidia-docker/wiki/Installation-(version-2.0)#prerequisites), you will need to ensure you have a supported Linux distribution and a supported NVIDIA GPU.

2. Install a [supported version of Docker Community Edition (CE)](https://github.com/NVIDIA/nvidia-docker/wiki/Frequently-Asked-Questions#which-docker-packages-are-supported).

3. Install the NVIDIA binary GPU driver, ensuring you use a version that meets [the minimum requirements for the CUDA version you intend to use](https://github.com/NVIDIA/nvidia-docker/wiki/CUDA#requirements) or at least version 361.93 if you don't intend to use CUDA.

4. [Install NVIDIA Docker version 2.0](https://github.com/NVIDIA/nvidia-docker/wiki/Installation-(version-2.0)#installing-version-20). (NVIDIA Docker version 1.0 is deprecated and will not work with recent container images.)

5. Pull one of the NVIDIA base container images from Docker Hub:
    
    - [nvidia/opengl](https://hub.docker.com/r/nvidia/opengl/) for OpenGL support
    - [nvidia/cudagl](https://hub.docker.com/r/nvidia/cudagl/) for OpenGL + CUDA support
    - [nvidia/cuda](https://hub.docker.com/r/nvidia/cuda/) for CUDA support
    - [nvidia/opencl](https://hub.docker.com/r/nvidia/opencl/) for OpenCL support

6. Build Unreal Engine container images based on the NVIDIA base images, either using [an existing source of images](../obtaining-images/image-sources) or by [writing your own custom Dockerfiles](../obtaining-images/write-your-own).


## Does NVIDIA Docker affect the compatibility of built container images?

By default, container images built on a system that has NVIDIA Docker installed **will be identical to container images built on a system without NVIDIA Docker.** This is because the default runc container runtime is used during the build process. The resulting container images can be run with GPU acceleration using the NVIDIA Docker runtime or without GPU acceleration using any other container runtime.

However, if you configure the Docker daemon to [use NVIDIA Docker as the default container runtime](https://github.com/NVIDIA/nvidia-docker/wiki/Advanced-topics#default-runtime) then any container images built by that Docker daemon [may be rendered non-portable](https://github.com/NVIDIA/nvidia-docker/wiki/Frequently-Asked-Questions#can-i-use-the-gpu-during-a-container-build-ie-docker-build). For this reason, it is strongly recommended that you do not reconfigure the default container runtime on hosts that will be used to build containers. (Using NVIDIA Docker as the default runtime on hosts that are only being used to run prebuilt containers is perfectly fine.)


## Is there any support for other platforms?

The NVIDIA Docker FAQ is extremely clear on this: [platforms other than Linux are not supported](https://github.com/NVIDIA/nvidia-docker/wiki/Frequently-Asked-Questions#platform-support). As stated previously, you can use a Docker client on another platform to connect to a Docker daemon that's using the NVIDIA Docker runtime, but the Docker daemon itself must be running under a Linux host system.

It is worth noting that there is [experimental support for GPU-accelerated Windows containers](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/gpu-acceleration) featured in preview builds of Docker running on the latest versions of Windows Server and Windows 10, but this functionality is not the same thing as the NVIDIA Docker runtime. This experimental support is restricted to running containers derived from the [mcr.microsoft.com/windows](https://hub.docker.com/_/microsoft-windows) base image and only provides hardware acceleration via DirectX. Other APIs such as OpenGL, OpenCL, CUDA and Vulkan are not supported.
