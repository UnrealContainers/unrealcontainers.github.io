---
title: Writing Dockerfiles
tagline: Write your own Dockerfiles to build Unreal Engine development container images.
source: "Custom Dockerfiles"
order: 5
---

{% capture _alert_content %}
Although writing your own Dockerfiles provides the greatest flexibility, it also involves understanding and addressing a number of technical complexities related to both the Unreal Engine itself and containers in general. Before writing your own Dockerfiles, we strongly recommended exploring the other [available sources of container images](./image-sources) and carefully studying the existing Dockerfiles.
{% endcapture %}
{% include alerts/warning.html title="This topic is not recommended for beginners." content=_alert_content %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Introduction

This page aims to provide general guidance for developers who wish to write their own Dockerfiles for building Unreal Engine [development container images](../concepts/image-types#development-images) that include the Engine build tools. This guidance takes the form of a set of brief tips, organised into platform-based sections below. Most of these tips are derived from experience gained during the development of the [ue4-docker project](./ue4-docker), and contain a number of references to the documentation and source code for that project. Nevertheless, the information presented on this page is designed to be implementation-agnostic, and applies to any Dockerfiles written for the Unreal Engine, irrespective of the infrastructure that is used to build them.

It is worth noting that the tips on this page relate only to writing Dockerfiles and not to configuring the environment or infrastructure that will be used to build the final container images. For details on environment configuration, see the [Environment Setup](../environments) section of the Unreal Containers community hub documentation.


## Platform-agnostic considerations

- Cloning a private Git repository during the container build process introduces a number of complications. Care must be taken to avoid inadvertently embedding your Git credentials in the layer history of the built images. This problem is exacerbated by the enormous size of the Unreal Engine GitHub repository, which makes copying the cloned repository to a new image via a [Docker multi-stage build](https://docs.docker.com/develop/develop-images/multistage-build/) quite slow and cumbersome. We recommend looking at the mechanisms that other [available sources of container images](./image-sources) utilise to address this issue.

- Some versions of the Unreal Engine have known issues in the `InstalledEngineBuild.xml` BuildGraph script that is used to create an Installed Build of the Engine. Under some circumstances it will be necessary to patch the XML data to fix these problems. Some known examples of such issues are:
  
  - Under Unreal Engine 4.19.x, the UnrealPak binary is not included in Installed Builds of the Engine under Linux. [^1]
  - Under Unreal Engine 4.20.x, native Windows Installed Builds require files from the Linux cross-compilation toolchain, even if the Linux target platform is disabled. [^2]


## Writing Dockerfiles for Linux containers

### General tips

- Make sure you derive your images from the [nvidia/opengl](https://hub.docker.com/r/nvidia/opengl/) base image, even if you don't plan on performing rendering using the NVIDIA Container Toolkit. If the Engine is built inside a plain Linux base image without any OpenGL libraries it will be built without OpenGL support, which means any Unreal projects built and packaged using the resulting image will be unable to perform rendering, even when run directly on a Linux host system outside of a container. *(See the first list item under [Tips for working with the NVIDIA Container Toolkit](#tips-for-working-with-the-nvidia-container-toolkit) below for a clarification of the number one question that people usually ask about this requirement.)*

- Both UnrealBuildTool and the Engine itself refuse to run as root under Linux. You will need to create a non-root user to run the build commands.

- The Unreal Engine started using a bundled compiler toolchain under Linux in version 4.20.0 [^3], and a bundled version of Mono in version 4.21.0. [^4] You will need to install clang and/or Mono in your container images when building older versions of the Unreal Engine.

- The bundled compiler toolchain is included automatically in Linux Installed Builds since version 4.22.1. You will need to copy the toolchain into Installed Builds manually when building older versions of the Unreal Engine.

- If you're encountering issues due to missing shared library files, you can use the [ldd](https://en.wikipedia.org/wiki/Ldd_(Unix)) and [apt-file](https://wiki.debian.org/apt-file) tools to identify missing libraries and the system packages that provide them.

### Tips for working with the NVIDIA Container Toolkit

- Container images built with [NVIDIA Container Toolkit](../concepts/nvidia-docker) support will still run correctly using any OCI-compatible container runtime, they just won't see any available GPU devices. Deriving an image from the [nvidia/opengl](https://hub.docker.com/r/nvidia/opengl/) or [nvidia/cudagl](https://hub.docker.com/r/nvidia/cudagl/) base images **does not** force you to use the NVIDIA Container Toolkit to run the resulting image. Built images will work happily under any runtime, irrespective of whether it is run under Linux, Windows, or macOS.

- GPU acceleration is not available during the build process. Although older versions of NVIDIA Docker allowed you to configure the Docker daemon to use the NVIDIA container runtime by default and thus enable build-time GPU access, [this was generally a bad idea because it could result in non-portable container images](https://github.com/NVIDIA/nvidia-docker/wiki/Frequently-Asked-Questions#can-i-use-the-gpu-during-a-container-build-ie-docker-build), and this option is not present in newer versions of the NVIDIA Container Toolkit. It is best to simply avoid running commands during the build process that require GPU access.

- If you want to enable X11 support in your containers then you will need to install the relevant system packages for the X11 runtime libraries. For Debian-based distributions, the required packages are as follows:
  
  - `libfontconfig1`
  - `libfreetype6`
  - `libglu1`
  - `libsm6`
  - `libxcomposite1`
  - `libxcursor1`
  - `libxi6`
  - `libxrandr2`
  - `libxrender1`
  - `libxss1`
  - `libxv1`
  - `x11-xkb-utils`
  - `xauth`
  - `xfonts-base`
  - `xkb-data`

- If you need to run automation tests for projects that use the Chromium Embedded Framework (CEF) by way of the [Web Browser Widget](https://docs.unrealengine.com/en-US/Engine/UMG/UserGuide/WidgetTypeReference/WebBrowser/index.html) then you'll need additional system packages on top of the X11 runtime libraries [^5]:
  
  - `libasound2`
  - `libatk1.0-0`
  - `libcairo2`
  - `libnss3`
  - `libnspr4`
  - `libpango-1.0-0`
  - `libpangocairo-1.0-0`

### PulseAudio support

If you want to enable audio support in your containers then the simplest option is to use PulseAudio. This will require the PulseAudio client libraries and server, which can be installed under Debian-based distributions via the `pulseaudio` system package. How you configure PulseAudio will depend on your desired use case, see the [Linux section](../concepts/audio-output#audio-output-in-linux-containers) of the [Audio output in containers](../concepts/audio-output) page for details of the available options.

## Writing Dockerfiles for Windows containers

- If it is feasible to do so, we recommend restricting support to Windows Server 2019 and newer. There are several reasons for this:
  
  - Windows Server 2019 introduced the [mcr.microsoft.com/windows](https://hub.docker.com/_/microsoft-windows) base image, which provides a number of DLL files required by the Unreal Engine. This base image can either be used directly or the relevant DLL files can be copied into the [mcr.microsoft.com/windows/servercore](https://hub.docker.com/_/microsoft-windows-servercore) base image to minimise the final image size. When building images for older versions of Windows Server, only the [mcr.microsoft.com/windows/servercore](https://hub.docker.com/_/microsoft-windows-servercore) base image is available and the required DLL files must be copied from another source, such as the host system.
  
  - Container images built for one version of Windows Server [are not compatible with other versions of Windows Server](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/version-compatibility), except through the use of Hyper-V isolation mode. Every additional version of Windows Server that you support multiplies the number of configurations that need to be built and tested, increasing maintenance requirements.
  
  - Older builds of Windows Server may contain bugs. For example, [the xcopy command does not work when running containers in process isolation mode](https://github.com/moby/moby/issues/38425) under certain builds of Windows Server 2016. Although these bugs can be avoided by ensuring your container hosts are kept fully up-to-date, they can be extremely difficult to debug in the case that a container host is inadvertently left unpatched.

- You will need to include the Visual Studio build tools in your container images. Microsoft provides [instructions and an example Dockerfile for installing the build tools in a container](https://docs.microsoft.com/en-us/visualstudio/install/build-tools-container).

- The Unreal Engine requires a number of DLL files that are not provided by the [mcr.microsoft.com/windows/servercore](https://hub.docker.com/_/microsoft-windows-servercore) base image or by an installation of the Visual Studio build tools. Without these DLL files, the Editor cannot be run to cook content or package Unreal projects. Some of these DLL files can be copied from the [mcr.microsoft.com/windows](https://hub.docker.com/_/microsoft-windows) base image under Windows Server 2019 and newer, whereas others must be downloaded and extracted from installer packages available online. As per [this article discussing the creation of the ue4-docker project](https://adamrehn.com/articles/building-docker-images-for-unreal-engine-4/), the list of required DLL files is as follows:
  
  - `MSVCR100.dll`
  - `xinput1_3.dll`
  - `D3DCompiler_43.dll`
  - `X3DAudio1_7.dll`
  - `XAPOFX1_5.dll`
  - `XAudio2_7.dll`
  - `dsound.dll`
  - `opengl32.dll`
  - `glu32.dll`
  - `vulkan-1.dll`

- If you need to run automation tests for projects that use the Chromium Embedded Framework (CEF) by way of the [Web Browser Widget](https://docs.unrealengine.com/en-US/Engine/UMG/UserGuide/WidgetTypeReference/WebBrowser/index.html) then you'll need additional DLL files on top of those listed above [^5]:
  
  - `BluetoothApis.dll`
  - `bthprops.cpl`
  - `dxva2.dll`
  - `mf.dll`
  - `mfplat.dll`
  - `mfreadwrite.dll`

- Creating an Installed Build of the Engine requires the [PDBCopy](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/pdbcopy) tool for stripping symbols from debug symbol files. As per the article referenced in the list item discussing required DLL files [^1], the PDBCopy executable can be obtained via the [windbg Chocolatey package](https://chocolatey.org/packages/windbg) and must be copied to the location where the Unreal Engine expects to find it, which varies based on Engine version and Visual Studio version.

- If an application running inside a Windows container attempts to use the Windows API to spawn a standard system UI component such as a message box or dialog box, the process will hang indefinitely. This issue is particularly relevant to the UnrealVersionSelector tool, which is invoked by `Setup.bat` to register the Engine installation and configure Windows Explorer shell integration. This invocation must be disabled in order to prevent the script from hanging indefinitely.

- Some versions of Visual Studio are quite memory-hungry and contain non-deterministically triggered bugs. It may be necessary to re-run failed builds in order to cope with these issues, which can be automated as either part of your Dockerfiles or by the infrastructure that builds your container images.

- If you're encountering issues due to missing DLL files, you can use the [dlldiag](https://github.com/adamrehn/dll-diagnostics) tool to help identify missing libraries.


## References

[^1]: [Article: "Building Docker images for Unreal Engine 4" by Adam Rehn](https://adamrehn.com/articles/building-docker-images-for-unreal-engine-4/)
[^2]: [ue4-docker source code on GitHub: file `patch-filters-xml.py`](https://github.com/adamrehn/ue4-docker/blob/master/ue4docker/dockerfiles/ue4-minimal/windows/patch-filters-xml.py)
[^3]: [Unreal Engine 4.20 Release Notes](https://docs.unrealengine.com/en-US/Builds/4_20)
[^4]: [Unreal Engine 4.21 Release Notes](https://docs.unrealengine.com/en-US/Builds/4_21)
[^5]: [ue4-docker GitHub issue #57: Failed to get CEF3 DLL handle](https://github.com/adamrehn/ue4-docker/issues/57)
