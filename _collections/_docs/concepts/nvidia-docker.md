---
title: NVIDIA Docker
tagline: What is the NVIDIA Container Toolkit and how can it be used to run Linux containers with full GPU acceleration?
quickstart: ["run"]
order: 2
---

{% capture _alert_content %}
- The [NVIDIA Container Toolkit](https://github.com/NVIDIA/nvidia-docker) (formerly known as NVIDIA Docker) allows containers to access full GPU acceleration.
- OpenGL, OpenCL and CUDA are supported for production use. Vulkan support is [currently in alpha](https://hub.docker.com/r/nvidia/vulkan).
- **This only works for Linux containers running on Linux host systems with NVIDIA GPUs.**
{% endcapture %}
{% include alerts/overview.html content=_alert_content %}


## Contents
{:.no_toc}

* TOC
{:toc}


## What is a container runtime?

{% include alerts/info.html content="This section is provided for readers who are interested in the underlying mechanisms that are used to run containers and how these mechanisms relate to GPU acceleration. If you're not interested in these details, feel free to skip ahead to the [What is the NVIDIA Container Toolkit?](#what-is-the-nvidia-container-toolkit) section." %}

As explained in [this article discussing container runtimes](https://www.ianlewis.org/en/container-runtimes-part-1-introduction-container-r), the term "container runtime" is used to refer to a variety of software components involved in the process of running containers. These components can be divided into two broad categories:

- **"Low-level" container runtimes:** these runtimes are the software component responsible for actually running containers using the host operating system's native isolation functionality. Low-level container runtimes are typically specific to a particular operating system or family of operating systems.

- **"High-level" container runtimes:** these runtimes are higher-level abstractions that typically sit on top of the low-level runtimes and provide additional convenience features. High-level container runtimes can achieve cross-platform portability by hooking into the appropriate low-level container runtime for any given host platform. A popular example of a cross-platform, high-level container runtime is [containerd](https://containerd.io/).

For the purposes of this discussion we are primarily interested in low-level container runtimes. Here are three examples of popular low-level container runtimes that can be used by Docker:

- [**runc**](https://github.com/opencontainers/runc): this is the default container runtime used by Docker to run containers under Linux host systems. The runc runtime interacts directly with the Linux kernel to run containers.

- [**runhcs**](https://github.com/Microsoft/hcsshim/tree/master/cmd/runhcs): this is the default container runtime used by Docker to run Windows containers under Windows host systems. The runhcs runtime interacts with the [Host Compute Service](https://techcommunity.microsoft.com/t5/Containers/Introducing-the-Host-Compute-Service-HCS/ba-p/382332), which in turn interacts directly with the Windows kernel to run containers. The runhcs runtime is a modified version of the runc runtime.

- [**nvidia-container-runtime**](https://github.com/NVIDIA/nvidia-container-runtime): this is the container runtime used by older versions of NVIDIA Docker to run containers with full GPU acceleration on NVIDIA hardware. The NVIDIA container runtime is a modified version of the runc runtime that automatically calls a [pre-start hook](https://github.com/opencontainers/runtime-spec/blob/master/config.md#prestart) provided by the underlying [libnvidia-container](https://github.com/NVIDIA/libnvidia-container) library, which interacts with the GPU drivers on the host system and exposes GPU devices to containers when they start. Starting in Docker 19.03, the Docker daemon supports calling this pre-start hook directly, removing the need for a separate runtime and allowing newer versions of the NVIDIA Container Toolkit to use the unmodified runc runtime.


## What is the NVIDIA Container Toolkit?

The [NVIDIA Container Toolkit](https://github.com/NVIDIA/nvidia-docker) (formerly known as NVIDIA Docker) is a plugin for the Docker daemon that allows containers to communicate with the GPU drivers on the host system, providing full access to all NVIDIA GPU devices. Older versions of NVIDIA Docker relied on a custom container runtime to invoke the necessary setup code when starting containers, whilst newer versions of the NVIDIA Container Toolkit rely on functionality added in Docker 19.03 that allows the Docker daemon to invoke the setup code directly without the need for a separate container runtime. 

Containers running with NVIDIA GPU acceleration can use a variety of hardware-accelerated APIs including OpenGL, OpenCL and NVIDIA CUDA. Vulkan support is [currently in alpha](https://hub.docker.com/r/nvidia/vulkan).

The NVIDIA Container Toolkit is designed specifically for Linux containers running on Linux host systems. The underlying code [does not support Windows containers](https://github.com/NVIDIA/nvidia-docker/wiki/Frequently-Asked-Questions#platform-support), nor can it be used when running Linux containers under Windows or macOS due to the fact that containers are run inside a Linux VM that does not have GPU access. However, Docker clients running under Windows and macOS can still be used to connect to a Docker daemon running under Linux with the NVIDIA Container Toolkit.


## What steps do I need to follow to make use of the NVIDIA Container Toolkit?

1. As per the [NVIDIA Container Toolkit prerequisites list](https://github.com/NVIDIA/nvidia-docker/wiki/Installation-(Native-GPU-Support)#prerequisites), you will need to ensure you have a supported Linux distribution and a supported NVIDIA GPU.

2. Install a [supported version of Docker Community Edition (CE) 19.03 or newer](https://github.com/NVIDIA/nvidia-docker/wiki/Frequently-Asked-Questions#which-docker-packages-are-supported).

3. Install the NVIDIA binary GPU driver, ensuring you use a version that meets [the minimum requirements for the CUDA version you intend to use](https://github.com/NVIDIA/nvidia-docker/wiki/CUDA#requirements) or at least version 361.93 if you don't intend to use CUDA.

4. [Install the NVIDIA Container Toolkit](https://github.com/NVIDIA/nvidia-docker/wiki/Installation-(Native-GPU-Support)#installing-gpu-support).

5. Pull one of the NVIDIA base container images from Docker Hub:
    
    - [nvidia/opengl](https://hub.docker.com/r/nvidia/opengl/) for OpenGL support
    - [nvidia/cudagl](https://hub.docker.com/r/nvidia/cudagl/) for OpenGL + CUDA support
    - [nvidia/cuda](https://hub.docker.com/r/nvidia/cuda/) for CUDA support
    - [nvidia/opencl](https://hub.docker.com/r/nvidia/opencl/) for OpenCL support

6. Build Unreal Engine container images based on the NVIDIA base images, either using [an existing source of images](../obtaining-images/image-sources) or by [writing your own custom Dockerfiles](../obtaining-images/write-your-own).


## Does the NVIDIA Container Toolkit affect the compatibility of built container images?

Container images built on a system that has the NVIDIA Container Toolkit installed **will be identical to container images built on a system without the NVIDIA Container Toolkit.** This is because GPU acceleration is not enabled during the build process. The resulting container images can be run with GPU acceleration using the NVIDIA Container Toolkit or without GPU acceleration using any OCI-compatible container runtime.

Older versions of NVIDIA Docker allowed the Docker daemon to [use NVIDIA Docker as the default container runtime](https://github.com/NVIDIA/nvidia-docker/wiki/Advanced-topics#default-runtime), which enabled GPU acceleration during image builds and meant that any container images built by that Docker daemon [could be rendered non-portable](https://github.com/NVIDIA/nvidia-docker/wiki/Frequently-Asked-Questions#can-i-use-the-gpu-during-a-container-build-ie-docker-build). For this reason, it was strongly recommended that you did not reconfigure the default container runtime on hosts that were used to build containers. This option is not present in newer versions of the NVIDIA Container Toolkit.


## Is there any support for other platforms?

The NVIDIA Container Toolkit FAQ is extremely clear on this: [platforms other than Linux are not supported](https://github.com/NVIDIA/nvidia-docker/wiki/Frequently-Asked-Questions#platform-support). As stated previously, you can use a Docker client on another platform to connect to a Docker daemon that's using the NVIDIA Container Toolkit, but the Docker daemon itself must be running under a Linux host system.

It is worth noting that there is [experimental support for GPU-accelerated Windows containers](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/gpu-acceleration) featured in newer versions of Docker running on the latest versions of Windows Server and Windows 10, but this functionality is not the same thing as the NVIDIA Container Toolkit. This experimental support is restricted to running containers derived from the [mcr.microsoft.com/windows](https://hub.docker.com/_/microsoft-windows) base image and only provides hardware acceleration via DirectX. Other APIs such as OpenGL, OpenCL, CUDA and Vulkan are not supported.
