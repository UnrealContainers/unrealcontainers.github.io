---
title: GPU acceleration in containers
tagline: How can GPU acceleration be used to perform rendering or computational tasks inside Linux and Windows containers?
order: 4
---

{% capture _alert_content %}
- Linux containers support all graphics APIs for NVIDIA GPUs using the [NVIDIA Container Toolkit](./nvidia-docker).

- Windows containers support DirectX-based graphics APIs for GPUs from all vendors using [native hardware acceleration support](./windows-containers#hardware-acceleration-support).
{% endcapture %}
{% include alerts/overview.html content=_alert_content %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Graphics APIs

Modern graphics hardware supports a variety of functionality including traditional rasterisation for rendering 2D or 3D graphics, raytracing for generating realisting lighting and reflections, video encoding and decoding, AI acceleration for machine learning tasks and [GPGPU](https://en.wikipedia.org/wiki/General-purpose_computing_on_graphics_processing_units) pipelines for general purpose computation. Each of these features are exposed to software through different Application Programming Interfaces (APIs) on different platforms, and understanding the purpose of these APIs is helpful when discussing GPU acceleration in the context of containers.

The following APIs are used for **rendering (rasterisation and raytracing)**:

- **OpenGL:** used for rasterisation on GPUs from all vendors under Windows and Linux. Does not support raytracing. It is worth noting that the Unreal Engine [dropped support for OpenGL on desktop platforms](https://docs.unrealengine.com/4.26/en-US/WhatsNew/Builds/ReleaseNotes/4_26/#removed:opengldesktoprendering) in version 4.26.0.

- **Vulkan:** used for rasterisation and raytracing on GPUs from all vendors under Windows and Linux. Raytracing support was [introduced in the Vulkan SDK version 1.2.162.0](https://www.khronos.org/news/press/vulkan-sdk-tools-and-drivers-are-ray-tracing-ready) in the form of the Vulkan Ray Tracing extensions. It is worth noting that the Unreal Engine does not yet include support for raytracing with Vulkan.

- **DirectX:** used for rasterisation and raytracing on GPUs from all vendors under Windows only. Raytracing support was [introduced in DirectX version 12](https://devblogs.microsoft.com/directx/announcing-microsoft-directx-raytracing/) in the form of the DirectX Raytracing (DXR) feature.

The following APIs are used for **encoding and decoding video**:

- **AMD Advanced Media Framework (AMF):** used for encoding and decoding video on AMD GPUs and APUs under Windows and Linux.

- **NVIDIA NVENC/NVDEC:** used for encoding (NVENC) and decoding (NVDEC) video on NVIDIA GPUs under Windows and Linux.

The following APIs are used for **machine learning tasks and general purpose computation**:

- **OpenCL:** used for GPGPU compute on GPUs from all vendors under Windows and Linux.

- **NVIDIA CUDA:** used for GPGPU compute on NVIDIA GPUs under Windows and Linux.

- **DirectML:** used for machine learning tasks on GPUs from all vendors under Windows only.

The set of graphics APIs that are available to software running inside a container determines which GPU features it can access. The availability of these APIs varies based on operating system and hardware vendor. Note that if you are unable to make use of GPU acceleration for a given configuration then it may still be possible to fall back to [software rendering](#software-rendering) as an alternative.


## NVIDIA GPU support for Linux containers

Linux containers can access full GPU acceleration on NVIDIA graphics hardware using the [NVIDIA Container Toolkit](./nvidia-docker), which is described in further detail on its dedicated page. The following graphics APIs are supported:

- OpenGL
- Vulkan
- NVIDIA NVENC/NVDEC
- OpenCL
- NVIDIA CUDA


## AMD GPU support for Linux containers

{% include alerts/info.html content="The authors of this documentation are still in the process of familiarising themselves with AMD GPU acceleration in Linux containers. This section will be updated when the relevant information has been gathered." %}


## GPU support for Windows containers

Windows containers can access limited GPU acceleration on graphics hardware from any vendor using [native hardware acceleration support](./windows-containers#hardware-acceleration-support), so long as the graphics drivers are compliant with Windows Display Driver Model (WDDM) version 2.5 or newer. The following graphics APIs are supported by default:

- DirectX ([including DirectX Raytracing](../../blog/offscreen-rendering-in-windows-containers/))
- DirectML

It is possible to enable support for additional graphics APIs as described in the blog post [Enabling vendor-specific graphics APIs in Windows containers](../../blog/enabling-vendor-specific-graphics-apis-in-windows-containers/). Please note that using graphics APIs other than DirectX inside Windows containers is not officially supported by Microsoft and is **not recommended for production use**. However, it does make it possible to access graphics APIs for encoding and decoding video, which allows Windows containers to be used for tasks that require hardware accelerated video encoding such as running [Pixel Streaming](../use-cases/pixel-streaming) applications.

For an example of a Windows container image suitable for use with GPU acceleration, see the blog post [Offscreen rendering in Windows containers](../../blog/offscreen-rendering-in-windows-containers/).


## Software rendering

Software rendering can be used as a fallback option in situations where access to graphics APIs for rendering is required but the real-time rendering performance of a GPU is not necessary.

### Software rendering in Linux containers

{% include alerts/info.html content="The authors of this documentation are still in the process of familiarising themselves with OpenGL and Vulkan software rendering in Linux containers. This section will be updated when the relevant information has been gathered." %}

### Software rendering in Windows containers

Under Windows, you can perform software rendering using the [Windows Advanced Rasterization Platform (WARP)](https://docs.microsoft.com/en-us/windows/win32/direct3darticles/directx-warp), which ships with DirectX. To use WARP when running the Unreal Editor or a packaged Unreal project, specify the `-warp` command-line flag. This flag is supported when using the Direct3D 12 rendering backend in Unreal Engine [version 4.14.0](https://github.com/EpicGames/UnrealEngine/blob/4.14.0-release/Engine/Source/Runtime/D3D12RHI/Private/D3D12RHIPrivate.h#L117-L122) and newer. (You can also use the `-dx12` command-line flag to force the use of the Direct3D 12 rendering backend for Unreal projects which are not configured to use it by default.)

**Note that you will still need to use a Windows container image that includes the DLL files for DirectX support in order to use WARP inside a container.** As a result, Windows container images suitable for use with [GPU acceleration](#gpu-support-for-windows-containers) are also suitable for use with software rendering.
