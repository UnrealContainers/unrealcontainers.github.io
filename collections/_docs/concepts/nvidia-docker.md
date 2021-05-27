---
title: NVIDIA Container Toolkit
tagline: What is the NVIDIA Container Toolkit and how can it be used to run Linux containers with full GPU acceleration?
order: 5
---

{% capture _alert_content %}
- The [NVIDIA Container Toolkit](https://github.com/NVIDIA/nvidia-docker) (formerly known as NVIDIA Docker) allows Linux containers to access full GPU acceleration.
- All graphics APIs are supported, including OpenGL, Vulkan, OpenCL, CUDA and NVENC/NVDEC.
- **This only works with NVIDIA GPUs for Linux containers running on Linux host systems or inside WSL2.**
{% endcapture %}
{% include alerts/overview.html content=_alert_content %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Overview

The [NVIDIA Container Toolkit](https://github.com/NVIDIA/nvidia-docker) (formerly known as NVIDIA Docker) is a library and accompanying set of tools for exposing NVIDIA graphics devices to Linux containers. It provides full GPU acceleration for containers running under Docker, containerd, LXC, Podman and Kubernetes. If you are interested in learning about the underlying architecture of the NVIDIA Container Toolkit then be sure to check out the [Architecture Overview](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/arch-overview.html) page of the official documentation.

Containers running with GPU acceleration have access to all supported graphics APIs on NVIDIA GPUs, including OpenGL, Vulkan, OpenCL, CUDA and NVENC/NVDEC. For details of what these APIs are used for, see the [GPU acceleration in containers](./gpu-acceleration) overview page.

The NVIDIA Container Toolkit is designed specifically for Linux containers running directly on Linux host systems or within Linux distributions under version 2 of the [Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/) (WSL2). The underlying code [does not support Windows containers](https://github.com/NVIDIA/nvidia-docker/wiki/Frequently-Asked-Questions#platform-support), nor can it be used when running Linux containers on macOS or Windows without WSL2 due to the fact that containers are run inside a Linux VM that does not have GPU access. However, Docker clients running under Windows and macOS can still be used to connect to a Docker daemon running under Linux with the NVIDIA Container Toolkit.

**For details of alternative options for other GPU vendors and operating systems, see the [GPU acceleration in containers](./gpu-acceleration) overview page.**


## Installation under Linux

1. As per the [supported platforms list](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#supported-platforms) and [prerequisites list](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#pre-requisites) from the [NVIDIA Container Toolkit Installation Guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html), you will need to ensure you have a supported Linux distribution and a supported NVIDIA GPU.

2. Install a [supported version of Docker Community Edition (CE) 18.09 or newer](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#container-runtimes).

3. Install the NVIDIA binary GPU driver, ensuring you use a version that meets [the minimum requirements for the CUDA version you intend to use](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html#cuda-major-component-versions) or at least version 418.81.07 if you don't intend to use CUDA.

4. Install the NVIDIA Container Toolkit by following [the instructions for your specific Linux distribution](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#docker).

5. If you would like to test out a specific graphics API, pull the revelant NVIDIA base container images from Docker Hub:
    
    - [nvidia/opengl](https://hub.docker.com/r/nvidia/opengl/) for OpenGL support
    - [nvidia/cuda](https://hub.docker.com/r/nvidia/cuda/) for CUDA support
    - [nvidia/cudagl](https://hub.docker.com/r/nvidia/cudagl/) for OpenGL + CUDA support
    - [nvidia/vulkan](https://hub.docker.com/r/nvidia/vulkan/) for OpenGL + Vulkan + CUDA support
    - [nvidia/opencl](https://hub.docker.com/r/nvidia/opencl/) for OpenCL support

6. If you intend to use the Unreal Engine with a [runtime container image](./image-types) then be sure to [choose a base image](../obtaining-images/image-sources#sources-of-unreal-engine-runtime-images) that is pre-configured to support the NVIDIA Container Toolkit.

6. If you intend to use the Unreal Engine with a [development container image](./image-types) then you will need to [choose an image source](../obtaining-images/image-sources#sources-of-unreal-engine-development-images) that supports the NVIDIA Container Toolkit. Depending on the graphics APIs that you are interested in using, it may be necessary to build a development image from source that extends the revelant NVIDIA base image.


## Installation under Windows with WSL2

{% include alerts/info.html content="The authors of this documentation are still in the process of familiarising themselves with the use of the NVIDIA Container Toolkit under WSL2. This section will be updated when the relevant information has been gathered." %}


## Container image compatibility

Container images built on a system that has the NVIDIA Container Toolkit installed **will be identical to container images built on a system without the NVIDIA Container Toolkit.** This is because GPU acceleration is not enabled during the build process. The resulting container images can be run with GPU acceleration using the NVIDIA Container Toolkit or without GPU acceleration using any OCI-compatible container runtime.

Older versions of NVIDIA Docker allowed the Docker daemon to [use NVIDIA Docker as the default container runtime](https://github.com/NVIDIA/nvidia-docker/wiki/Advanced-topics#default-runtime), which enabled GPU acceleration during image builds and meant that any container images built by that Docker daemon [could be rendered non-portable](https://github.com/NVIDIA/nvidia-docker/wiki/Frequently-Asked-Questions#can-i-use-the-gpu-during-a-container-build-ie-docker-build). For this reason, it was strongly recommended that you did not reconfigure the default container runtime on hosts that were used to build containers. This option is not present in newer versions of the NVIDIA Container Toolkit.
