---
title: Cloud Rendering (Deprecated)
tagline: "Note: this overview page has been deprecated and is being split into other, more specific use cases."
order: 20
---

{% capture _alert_content %}
This page used to serve as an overview of several related use cases, which are now being split into separate pages with their own specific focus. The following pages cover use cases that were previously grouped by this page:

- [Pixel Streaming](./pixel-streaming)
- [Rendering Linear Media](./linear-media)
- The rendering-related parts of [Microservices](./microservices)
- The rendering-related parts of [AI and Machine Learning](./machine-learning)

The content below (which is now outdated in several places) will remain unchanged during the transition process, and will then be removed once everything has been relocated to the appropriate pages.
{% endcapture %}
{% include alerts/warning.html title="This page has been deprecated." content=_alert_content %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Overview

The [NVIDIA Container Toolkit](../concepts/nvidia-docker) allows Unreal Engine containers to run in the cloud with full GPU acceleration, facilitating server-side 2D or 3D rendering. GPU-accelerated containers augment existing cloud rendering workflows with all of the advantages inherent to containerisation, including increased density and compatibility with container orchestration technologies. Rendering results can be captured, stored, or streamed to remote devices in much the same manner as when performing cloud rendering inside VMs. Container-based cloud rendering can also be combined with existing RPC frameworks to create GPU-accelerated [microservices powered by the Unreal Engine](./microservices) or provide training data for [machine learning models](./machine-learning).


## Key considerations

- Because the NVIDIA Container Toolkit only works with Linux containers running under Linux host systems, cloud rendering cannot be performed inside [Windows containers](../concepts/windows-containers). Windows-based cloud rendering must be run inside VMs rather than containers.

- The NVIDIA Container Toolkit supports both OpenGL and Vulkan rendering. You will need to select a container base image that supports the rendering API you wish to use, either from the [official base images that NVIDIA makes available](https://hub.docker.com/u/nvidia), or from [the list of preconfigured base images available](../obtaining-images/image-sources#sources-of-base-images-for-running-packaged-unreal-projects) that are specifically designed for running packaged Unreal projects.

- When using OpenGL, the Unreal Engine will default to offscreen rendering when running in an environment without an X11 server, such as a Docker container. (When using Vulkan, offscreen rendering must be manually enabled by specifying the `-RenderOffscreen` flag, irrespective of whether an X11 server is present or not.) If your project requires X11 support then you will need to use a container image that includes the X11 runtime libraries and bind-mount the X11 socket from the host system using the flags `-v/tmp/.X11-unix:/tmp/.X11-unix:rw -e DISPLAY`.

- By default, containers will not have access to the audio devices from the host system and so the Unreal Engine's audio output will be disabled. If you are using a [preconfigured base image](../obtaining-images/image-sources#sources-of-base-images-for-running-packaged-unreal-projects) that includes PulseAudio support and has both the default [autospawning behaviour](https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/Running/#autospawning) enabled and the [`module-always-sink` module](https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/Modules/#module-always-sink) loaded then audio output will function correctly inside the container, since the PulseAudio server will automatically start when the Unreal Engine initialises audio output and will supply a virtual output device that facilitates audio output in the absence of any physical audio devices. If you need audio output to be routed through the host system's audio devices instead (e.g. for debugging purposes) then you will need to bind-mount the PulseAudio socket from the host system using the flag `"-v/run/user/$UID/pulse:/run/user/1000/pulse"` and use a container image configured to communicate with the PulseAudio server over this bind-mounted socket. Note that this will not work for the root user, so you will need to run the command as a non-root user as described by the [Post-installation steps for Linux](https://docs.docker.com/install/linux/linux-postinstall/) page of the Docker documentation.

- X11 support requires specific configuration of the host system when performing cloud rendering inside Linux VMs without desktop environments installed. See the pages in the [Environment Setup](../environments/) category of the documentation for details on performing these configuration steps.


## Implementation guidelines

### Base image selection

Packaged Unreal projects do not require the Engine Tools and can run in any container image that includes the required runtime libraries. As discussed in the [Sources of base images for running packaged Unreal projects](../obtaining-images/image-sources#sources-of-base-images-for-running-packaged-unreal-projects) section of the Available Image Sources page, container images that will perform rendering must be derived from the [nvidia/opengl](https://hub.docker.com/r/nvidia/opengl/), [nvidia/vulkan](https://hub.docker.com/r/nvidia/vulkan/) or [nvidia/cudagl](https://hub.docker.com/r/nvidia/cudagl/) base images. Beyond this single requirement, developers are free to design their runtime base images as they see fit.

#### Using a preconfigured base image

There are a number of pre-configured base images maintained by the community that can be used to quickly create container images suitable for cloud rendering workloads. These images often provide multiple variants to cater to common cloud rendering scenarios. See [this section of the Available Image Sources page](../obtaining-images/image-sources#sources-of-base-images-for-running-packaged-unreal-projects) for a list of available preconfigured base images.

#### Writing a custom Dockerfile

Writing your own Dockerfiles for building runtime base images is quite straightforward:

- **Select the appropriate NVIDIA base image for your intended usage scenario.** Choose the [nvidia/cudagl](https://hub.docker.com/r/nvidia/cudagl/) image if CUDA support is required, or the [nvidia/opengl](https://hub.docker.com/r/nvidia/opengl/) image if CUDA support is not required. (The [nvidia/vulkan](https://hub.docker.com/r/nvidia/vulkan/) image does not currently include any variants without CUDA support, so if you need Vulkan support but want to avoid the additional size overheads of CUDA then you'll need to use the [nvidia/opengl](https://hub.docker.com/r/nvidia/opengl/) image and add Vulkan support in your Dockerfile.)

- **Add PulseAudio support if audio output is required and/or X11 support if X11 window creation is required.** Instructions for adding PulseAudio support and X11 support can be found in the [PulseAudio support](../obtaining-images/write-your-own#pulseaudio-support) and [Tips for working with the NVIDIA Container Toolkit](../obtaining-images/write-your-own#tips-for-working-with-the-nvidia-container-toolkit) sections of the custom Dockerfile guide.

- **Add any additional runtime dependencies.** Simply install any other required packages in exactly the same manner as when writing Dockerfiles for traditional (non GPU-accelerated) Linux container images.

### Building container images for deployment

#### Copying packaged projects into container images from the host system

There is no inherent need to use Unreal Engine containers to build and package projects that will be deployed using Unreal Engine containers. If you've built and packaged an Unreal project for Linux on the host system then you can simply copy the packaged files into a container image using a `COPY` directive in your Dockerfile.

#### Creating container images as part of the build process for projects

If you're already making use of Unreal Engine containers as part of a [continuous integration](./continuous-integration) workflow then you can use a [Docker multi-stage build](https://docs.docker.com/develop/develop-images/multistage-build/) to build your runtime container images as part of the overarching build process. See the [Building container images for deployment](../use-cases/continuous-integration#building-container-images-for-deployment) section of the CI/CD page for more details on this option.

### Offscreen rendering flags

The flags required to enable offscreen rendering vary based on the version of the Unreal Engine you are using:

- **Unreal Engine versions 4.24 and older** only support offscreen rendering using OpenGL, and will default to offscreen rendering if an X11 server is not detected. If you're using a version of the Unreal Engine that defaults to Vulkan rendering in a container that also supports Vulkan then you will need to specify the `-opengl4` flag when running your packaged Unreal project to ensure it uses OpenGL. If an X11 server is present and you want to force offscreen rendering then you will need to set the `DISPLAY` environment variable to an empty value.

- **Unreal Engine versions 4.25 and newer** support offscreen rendering using both OpenGL and Vulkan. When using OpenGL, the flags and environment variables from the previous list item apply. When using Vulkan, you will need to specify the `-RenderOffscreen` flag when running your packaged Unreal project to enable offscreen rendering, since it is not enabled automatically in the absence of an X11 server.

### Capturing rendering output

There are a variety of different mechanisms that can be utilised to capture cloud rendering output. Each mechanism is suited to different scenarios and offers different advantages and limitations:

- The official implementation of the [Pixel Streaming](https://docs.unrealengine.com/en-us/Platforms/PixelStreaming) feature introduced in Unreal Engine 4.21.0 is currently Windows-only, and is therefore not compatible with container-based cloud rendering. **However, [Linux support for Pixel Streaming](https://adamrehn.com/articles/pixel-streaming-in-linux-containers/) is available**, and this alternative implementation is compatible with container-based cloud rendering by design.

- Traditional Linux video capture mechanisms can be used inside containers, but typically require additional configuration and sometimes even additional container security permissions. These techniques also scale poorly to large numbers of containers performing cloud rendering within a single host system, **and are not recommended.**

- The community-maintained [UE4Capture](https://github.com/adamrehn/UE4Capture) plugin is quite similar to the official Pixel Streaming functionality, albeit with full cross-platform support. The [ue4-docker project](../obtaining-images/ue4-docker) includes UE4Capture in the [ue4-full container image](https://docs.adamrehn.com/ue4-docker/building-images/available-container-images#ue4-full). It is worth noting that UE4Capture does not currently feature support for relaying user input, and is limited to capturing video and audio output from the Unreal Engine only.

- The framebuffer can be captured programmatically from within the Unreal Engine itself, either via the [HighResShot console command](https://docs.unrealengine.com/en-us/Engine/Basics/Screenshots) or the classes from the [MovieSceneCapture module](https://api.unrealengine.com/INT/API/Runtime/MovieSceneCapture/index.html). Audio output can be similarly captured through the AudioMixer system by registering a submix buffer listener that implements the [ISubmixBufferListener interface](https://api.unrealengine.com/INT/API/Runtime/Engine/ISubmixBufferListener/index.html) that was introduced in Unreal Engine 4.20.0.


## Related media

{% include related-items/media.html url=page.url %}


## Related repositories

{% include related-items/repositories.html url=page.url %}
