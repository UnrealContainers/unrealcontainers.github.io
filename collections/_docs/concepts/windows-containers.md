---
title: Windows Containers
tagline: How are Windows containers different to Linux containers and how can they be used most effectively?
order: 2
---

{% capture _alert_content %}
- Despite their similarities to Linux containers, Windows containers introduce additional complexities that must be understood in order to use them in an optimal manner.

- Process isolation mode should be strongly preferred over Hyper-V isolation mode when building and running Unreal Engine containers.

- Choose the version of Windows Server that you run on your container hosts carefully to avoid excessive maintenance overheads.

{% endcapture %}
{% include alerts/overview.html content=_alert_content %}

Windows containers are a far newer technology than their more mature Linux counterparts, having first been introduced in Windows Server 2016. The manner in which Windows container images interact with the host system kernel brings with it additional mechanisms and associated complexities which are not present when working with Linux containers. This page provides an overview of the key differences between Windows containers and Linux containers, and provides guidance on how to achieve optimal performance when building and running Windows containers.


## Contents
{:.no_toc}

* TOC
{:toc}


## Windows container architecture

The [Windows container platform page](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/containerd) provides an overview of the key architectural differences between containers running under Windows hosts and those running under Linux hosts. Further technical detail is provided in [this DockerCon presentation discussing the initial implementation of container support in Windows](https://www.youtube.com/watch?v=85nCF5S8Qok). The key architectural differences are summarised below:

- Under Linux, low-level container runtimes interact directly with the kernel to run containers. Under Windows, a system service called the [Host Compute Service (HCS)](https://techcommunity.microsoft.com/t5/Containers/Introducing-the-Host-Compute-Service-HCS/ba-p/382332) acts as the public interface to container functionality and abstracts the underlying kernel implementation. [^1] This allows Windows to provide features such as [multiple isolation modes for Windows containers](#isolation-modes) and support for Linux containers in a manner that is largely transparent to user-facing applications such as Docker.

- Under Linux, Docker uses union filesystems such as [OverlayFS](https://docs.docker.com/storage/storagedriver/overlayfs-driver/) to support filesystem layering in a performant manner. The complexities of the NTFS filesystem make it infeasible to provide full union filesystem functionality, so Windows containers use a hybrid implementation that behaves as closely to a union filesystem as possible. [^2] As a result, filesystem layer operations for Windows containers are slower than their Linux counterparts.

- Under Linux, a container includes only the processes that a user starts in that container. Due to the service-based architecture that is inherent to the way Windows is implemented, Windows containers include a number of standard system services in addition to user-started processes. [^2] To allow developers the ability to balance image size against available features, [multiple Windows base images](#windows-base-image-variants) are provided that are differentiated by the libraries and system services that each includes.


## Kernel version compatibility

Due to the excellent backward-compatibility of the Linux kernel, Linux containers encapsulating older distributions will continue to work correctly with newer kernel versions. Newer distributions will also often continue to function correctly on older kernel versions, although this will not be the case if the software inside the container relies on newer kernel features that are not present in the older kernel. As a result, the Linux host system needs only to maintain an up-to-date kernel version to ensure maximum compatibility with a wide range of Linux container images [^3].

This is not the case for Windows containers. Container images based on older Windows versions will not run on newer host kernel versions, nor will newer containers run on older host kernel versions [^4]. Microsoft maintains a [compatibility table detailing supported host/container configurations](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/version-compatibility), accompanied by a detailed discussion of how Windows container version compatibility works. As can be seen in the compatibility table, older container images can actually be run on newer host systems, but only by utilising a mechanism called Hyper-V isolation mode, which is described in the next section. It is worth noting that [work is underway](https://github.com/microsoft/Windows-Containers/issues/36) to improve kernel version compatibility in newer versions of Windows.


## Isolation modes

Windows containers can run in one of two "isolation modes":

- **Process isolation mode (also referred to as *Windows Server containers*):** this isolation mode works in much the same manner as traditional Linux containers, whereby the containers interact directly with the kernel of the host system that is running the Docker daemon. Process isolation mode is only available under Windows Server and [Windows 10 version 1809 or newer](https://docs.microsoft.com/en-us/virtualization/windowscontainers/about/faq#can-i-run-windows-containers-in-process-isolated-mode-on-windows-10-) and requires the Windows version of the container image and the host system to match as per the requirements discussed in the [Kernel version compatibility](#kernel-version-compatibility) section. When running older container images or running under Windows 10 version 1803 or older, Hyper-V isolation mode must be used.

- **Hyper-V isolation mode (also referred to as *Hyper-V containers*):** this isolation mode utilises Hyper-V virtual machines to run containers. When a container is started, a Hyper-V VM containing the appropriate kernel version is launched and the container then runs inside the VM. In this mode, the container interacts with the virtualised kernel instead of the host system kernel [^5]. Hyper-V isolation mode is available under both Windows Server and Windows 10, and was the only supported isolation mode under Windows 10 [prior to Windows 10 version 1809](https://docs.microsoft.com/en-us/virtualization/windowscontainers/about/faq#can-i-run-windows-containers-in-process-isolated-mode-on-windows-10-). It is worth noting that although Hyper-V isolation allows older containers to run on systems with newer kernel versions, it does **not** allow newer containers to run on host systems with older kernel versions, so it is necessary to keep the host system up to date with the latest version of Windows Server or Windows 10 in order to ensure compatibility with all Windows container images.

Although Hyper-V isolation mode offers the convenience of improved compatibility by decoupling the host kernel from running containers, the use of Hyper-V VMs introduces additional complexities and associated issues. These issues are discussed in the next section.

{% capture _alert_content %}
[HostProcess containers](https://github.com/kubernetes/enhancements/tree/master/keps/sig-windows/1981-windows-privileged-container-support) are a new type of Windows container that will be supported in upcoming releases of containerd and Kubernetes. The Microsoft documentation has not yet been updated to include information about HostProcess containers, so it is unclear whether this will be classified as a third isolation mode or a separate category entirely. This page will be updated to reflect Microsoft's decision once their documentation is released.
{% endcapture %}
{% include alerts/info.html title="What about HostProcess containers?" content=_alert_content %}


## Hyper-V isolation mode issues

There are a number of issues present in the current implementation of Hyper-V isolation mode that significantly hinder its suitability for use in production systems:

- **Reduced performance:** despite utilising specially-designed Hyper-V VMs that contain only the minimum required components for running containers, the overheads associated with launching and communicating with virtual machines still result in a noticeable reduction in performance when compared to containers running in process isolation mode. Container startup times are increased, and all disk I/O operations are slower, which is particularly problematic when building extremely large container images such as Unreal Engine images that include the Engine Tools. The spawned Hyper-V VMs will also default to only using two CPU cores unless the `--cpu-count` flag is specified to override this behaviour [^6].

- **Inability to control pagefile size:** since memory allocation requests are fulfilled by the kernel that a container is interacting with, containers running in process isolation mode will use the pagefile settings of the host system. However, there is currently no mechanism exposed by Hyper-V isolation mode to configure the pagefile size that is used within the spawned Hyper-V VMs.

- **Inability to control system locale:** system locale queries are fulfilled by the host kernel in the same manner as memory allocation requests, which means containers always use the locale settings of the host system. There is currently no mechanism exposed by Hyper-V isolation mode to configure the system locale that is used within spawned Hyper-V VMs.

- **Bind-mounted path incompatibilities:** the underlying paths associated with bind-mounted directories inside Hyper-V containers have been observed to cause issues for a variety of build tools, including UnrealBuildTool and CMake. This necessitates a workaround whereby source files are copied into a temporary location within the container filesystem so that they can be compiled, and the built binaries are subsequently copied back to the bind-mounted directory. Due to the reduced disk I/O performance of Hyper-V isolation mode, these additional copy operations incur noticeable overheads when compared to containers running in process isolation mode.

- **Incompatibility with hardware acceleration:** the [experimental hardware acceleration support](#hardware-acceleration-support) featured in recent versions of Docker is compatible only with process isolation mode, and cannot be used with containers running in Hyper-V isolation mode.

The implementation of Hyper-V isolation mode under older versions of Windows also suffered from additional issues that have subsequently been fixed in modern releases:

- **Container timeout errors:** containers running in Hyper-V isolation mode under older versions of Windows would sometimes fail to start (or to commit their filesystem layers in a build) and emit the error [hcsshim: timeout waiting for notification extra info](https://github.com/Microsoft/hcsshim/issues/152). What made this error particularly problematic was that it appeared to persist once triggered, and would typically prevent any further containers from starting until it had been dealt with. This issue was fixed in Windows Server 2019 and Windows 10 version 1809. [^7]


## Windows base image variants

Microsoft provides three primary variants of the Windows base image:

- [**mcr.microsoft.com/windows**](https://hub.docker.com/_/microsoft-windows): this is the full Windows client base image, containing almost all of the libraries and services that are included in a desktop installation of Windows 10. This base image was introduced in Windows Server 2019 / Windows Server version 1809.

- [**mcr.microsoft.com/windows/servercore**](https://hub.docker.com/_/microsoft-windows-servercore): this is the Windows Server Core base image, and is largely equivalent to a host installation of Windows Server Core in terms of the libraries and services that are included, albeit with a handful of features and roles removed to save space.

- [**mcr.microsoft.com/windows/nanoserver**](https://hub.docker.com/_/microsoft-windows-nanoserver): this is the Nano Server base image, which provides the smallest image size and the fewest number of included libraries and services. Although Nano Server was originally a host installation option for Windows Server 2016, it has been available exclusively as a container base image since Windows Server version 1709. [^8]

{% include alerts/info.html content="Windows Server 2022 will [introduce a fourth base image](https://techcommunity.microsoft.com/t5/containers/announcing-a-new-windows-server-container-image-preview/ba-p/2304897) called **mcr.microsoft.com/windows/server**, which is equivalent to a full installation of Windows Server with the [Desktop Experience](https://docs.microsoft.com/en-us/windows-server/get-started/getting-started-with-server-with-desktop-experience) enabled. The list above will be updated with the relevant details once Windows Server 2022 is released." %}

All Windows container images must derive either directly or indirectly from one of these base image variants. (The `FROM scratch` directive that can be used to [create Linux base images](https://docs.docker.com/develop/develop-images/baseimages/) is not supported for Windows containers, although there may be [an equivalent supported for HostProcess containers](https://github.com/kubernetes/enhancements/tree/master/keps/sig-windows/1981-windows-privileged-container-support#container-images).) It is also worth noting that the filesystem layers for each of the Windows base images are marked as [foreign layers](https://docs.microsoft.com/en-us/virtualization/windowscontainers/about/faq#how-do-i-make-my-container-images-available-on-air-gapped-machines-), which means they are not stored in private container registries by default and are instead pulled directly from the [Microsoft Container Registry](https://azure.microsoft.com/en-us/blog/microsoft-syndicates-container-catalog/).


## Hardware acceleration support

{% include alerts/info.html content="For information about GPU acceleration specifically, see [GPU support for Windows containers](./gpu-acceleration#gpu-support-for-windows-containers)." %}

Docker versions 19.03 and newer running on Windows Server 2019 or newer and Windows 10 version 1809 or newer can utilise experimental support for [accessing hardware devices inside Windows containers](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/hardware-devices-in-containers) by specifying the `--device` flag, in a similar manner to how this flag is [currently used to expose hardware devices to Linux containers](https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities). The experimental device access functionality includes special treatment of GPU devices, with support for [hardware acceleration using the DirectX API](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/gpu-acceleration). It is important to note that this functionality is not the same as that provided by the [NVIDIA Container Toolkit](./nvidia-docker) for Linux containers, which supports OpenGL, OpenCL, Vulkan and NVIDIA CUDA.

Official support for hardware acceleration is currently limited to containers derived from the [mcr.microsoft.com/windows](https://hub.docker.com/_/microsoft-windows) base image that are running in process isolation mode, although testing performed by the authors of this documentation has indicated that GPU acceleration will also function correctly in other base images if the necessary DLL files are copied over from the Windows client base image. *(Note that the announcement for the upcoming **mcr.microsoft.com/windows/server** base image [indicates](https://techcommunity.microsoft.com/t5/containers/announcing-a-new-windows-server-container-image-preview/ba-p/2304897#toc-hId--1609476460) that hardware acceleration will also be officially supported for that image.)*

Hardware acceleration is **not supported for containers running in Hyper-V isolation mode**, and testing has confirmed that this is indeed a genuine technical limitation.


## Recommendations for an optimal experience

- **Prefer process isolation mode over Hyper-V isolation mode when using Unreal Engine containers**: the performance limitations associated with containers running in Hyper-V isolation mode are greatly exacerbated when working with large, complex container images such as Unreal Engine containers. Hyper-V isolation mode is particularly ill-suited to building Unreal Engine development images that include the Engine Tools, due to an inability to configure virtual memory settings and poor filesystem performance when working with extremely large filesystem layers. Note that process isolation mode requires the use of Windows Server or [Windows 10 version 1809 or newer](https://docs.microsoft.com/en-us/virtualization/windowscontainers/about/faq#can-i-run-windows-containers-in-process-isolated-mode-on-windows-10-).

- **Prefer Windows Server 2019 or newer when possible:** Windows Server 2019 [introduced a number of new features for Windows containers](https://docs.microsoft.com/en-us/windows-server/get-started-19/whats-new-19#application-platform), including support for the [mcr.microsoft.com/windows](https://hub.docker.com/_/microsoft-windows) base image, which contains a number of DLL files that are required by the Unreal Engine. It also fixed a number of bugs that were present in earlier versions of Windows Server and expanded the architecture of the Windows container stack with new components. [^1]

- **Do not use Windows Server versions 1903 or 1909, or Windows 10 versions 1903 or 1909 to build Windows container images with Docker version 19.03.5 or older:** there is a known bug in Windows Server versions 1903 and 1909 and Windows 10 versions 1903 and 1909 that [prevents Docker from building images larger than the default limit of 20GB](https://github.com/docker/for-win/issues/4100). A workaround for this bug was [introduced in Docker version 19.03.6](https://github.com/docker/engine/pull/429).

- **Limit the number of Windows Server versions supported on container hosts:** because container images built for one version of Windows Server are not compatible with other versions of Windows Server (except through the use of Hyper-V isolation mode), every additional version of Windows Server that you support on your container hosts multiplies the number of container image configurations that need to be built and tested, significantly increasing maintenance requirements. It is recommended that you limit support to either:
    - Just the current release in the [Long-Term Servicing Channel (LTSC)](https://docs.microsoft.com/en-us/windows-server/get-started-19/servicing-channels-19#long-term-servicing-channel-ltsc) (currently Windows Server 2019), **or**
    - Both the current LTSC release and the current release in the [Semi-Annual Channel (SAC)](https://docs.microsoft.com/en-us/windows-server/get-started-19/servicing-channels-19#semi-annual-channel) (currently Windows Server version 1909, although be sure to use the latest version of Docker to avoid the issues mentioned above.)


## References

[^1]: [Microsoft Docs: Windows container platform](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/containerd)
[^2]: [Windows Server & Docker - The Internals Behind Bringing Docker & Containers to Windows - Black Belt](https://www.youtube.com/watch?v=85nCF5S8Qok)
[^3]: [LXD Issue #4484: Clarification on running older Ubuntu versions on a newer kernel](https://github.com/lxc/lxd/issues/4484)
[^4]: [Microsoft Docs: Windows Container Version Compatibility](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/version-compatibility)
[^5]: [Microsoft Docs: Hyper-V Isolation](https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/hyperv-container)
[^6]: [Docker for Windows Issue #1877: Hyper-V isolation uses only 2 cores](https://github.com/docker/for-win/issues/1877)
[^7]: [hcsshim Issue #152: hcsshim: timeout waiting for notification extra info (comment by jhowardmsft)](https://github.com/Microsoft/hcsshim/issues/152#issuecomment-469530581)
[^8]: [Microsoft Docs: Install Nano Server](https://docs.microsoft.com/en-us/windows-server/get-started/getting-started-with-nano-server)
