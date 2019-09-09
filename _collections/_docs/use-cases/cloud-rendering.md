---
title: Cloud Rendering
tagline: Perform 2D or 3D rendering in the cloud with full GPU acceleration via NVIDIA Docker.
quickstart: ["run"]
order: 2
---

{% capture _alert_content %}
- Base container image(s) that [support running packaged Unreal projects with GPU acceleration via NVIDIA Docker](../obtaining-images/image-sources)
- A Linux environment [configured for running containers via NVIDIA Docker](../environments)
{% endcapture %}
{% include alerts/required.html content=_alert_content %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Overview

The [NVIDIA Docker runtime](../concepts/nvidia-docker) allows Unreal Engine containers to run in the cloud with full GPU acceleration, facilitating server-side 2D or 3D rendering. GPU-accelerated containers augment existing cloud rendering workflows with all of the advantages inherent to containerisation, including increased density and compatibility with container orchestration technologies. Rendering results can be captured, stored, or streamed to remote devices in much the same manner as when performing cloud rendering inside VMs. Container-based cloud rendering can also be combined with existing RPC frameworks to create GPU-accelerated [microservices powered by the Unreal Engine](./microservices) or provide training data for [machine learning models](./machine-learning).


## Key considerations

- Because NVIDIA Docker only works with Linux containers running under Linux host systems, cloud rendering cannot be performed inside [Windows containers](../concepts/windows-containers). Windows-based cloud rendering must be run inside VMs rather than containers.

- NVIDIA Docker currently only supports OpenGL rendering. Vulkan support [has been promised as a future addition](https://github.com/NVIDIA/nvidia-docker/wiki/Frequently-Asked-Questions#is-vulkan-supported).

- The Unreal Engine will default to offscreen rendering when running in an environment without an X11 server, such as a Docker container. If your project requires X11 support then you will need to use a container image that includes the X11 runtime libraries and bind-mount the X11 socket from the host system using the flags `-v/tmp/.X11-unix:/tmp/.X11-unix:rw -e DISPLAY`.

- By default, containers will not have access to the audio devices from the host system and so audio output will be disabled. When using container images that include PulseAudio support, you can enable audio output by bind-mounting the PulseAudio socket from the host system using the flag `"-v/run/user/$UID/pulse:/run/user/1000/pulse"`. Note that this will not work for the root user, so you will need to run the command as a non-root user as described by the [Post-installation steps for Linux](https://docs.docker.com/install/linux/linux-postinstall/) page of the Docker documentation.

- Both X11 support and PulseAudio support require specific configuration of the host system when performing cloud rendering inside Linux VMs without desktop environments installed. See the pages in the [Environment Setup](../environments/) category of the documentation for details on performing these configuration steps.


## Implementation guidelines

### Base image selection

Packaged Unreal projects do not require the Engine Tools and can run in any container image that includes the required runtime libraries. As discussed in the [Sources of base images for running packaged Unreal projects](../obtaining-images/image-sources#sources-of-base-images-for-running-packaged-unreal-projects) section of the Available Image Sources page, container images that will perform rendering must be derived from the [nvidia/opengl](https://hub.docker.com/r/nvidia/opengl/) or [nvidia/cudagl](https://hub.docker.com/r/nvidia/cudagl/) base images. Beyond this single requirement, developers are free to design their runtime base images as they see fit.

#### Using a preconfigured base image

There are a number of pre-configured base images maintained by the community that can be used to quickly create container images suitable for cloud rendering workloads. These images often provide multiple variants to cater to common cloud rendering scenarios. See [this section of the Available Image Sources page](../obtaining-images/image-sources#sources-of-base-images-for-running-packaged-unreal-projects) for a list of available preconfigured base images.

#### Writing a custom Dockerfile

Writing your own Dockerfiles for building runtime base images is quite straightforward:

- **Select the appropriate NVIDIA base image for your intended usage scenario.** Choose the [nvidia/cudagl](https://hub.docker.com/r/nvidia/cudagl/) image if CUDA support is required, or the [nvidia/opengl](https://hub.docker.com/r/nvidia/opengl/) image if CUDA support is not required.

- **Add PulseAudio support if audio output is required and/or X11 support if X11 window creation is required.** Instructions for adding PulseAudio support and X11 support can be found in the [Tips for working with NVIDIA Docker](../obtaining-images/write-your-own#tips-for-working-with-nvidia-docker) section of the custom Dockerfile guide.

- **Add any additional runtime dependencies.** Simply install any other required packages in exactly the same manner as when writing Dockerfiles for traditional (non GPU-accelerated) Linux container images.

### Building container images for deployment

#### Copying packaged projects into container images from the host system

There is no inherent need to use Unreal Engine containers to build and package projects that will be deployed using Unreal Engine containers. If you've built and packaged an Unreal project for Linux on the host system then you can simply copy the packaged files into a container image using a `COPY` directive in your Dockerfile.

#### Creating container images as part of the build process for projects

If you're already making use of Unreal Engine containers as part of a [continuous integration](./continuous-integration) workflow then you can use a [Docker multi-stage build](https://docs.docker.com/develop/develop-images/multistage-build/) to build your runtime container images as part of the overarching build process. See the [Building container images for deployment](../use-cases/continuous-integration#building-container-images-for-deployment) section of the CI/CD page for more details on this option.

### Capturing rendering output

There are a variety of different mechanisms that can be utilised to capture cloud rendering output. Each mechanism is suited to different scenarios and offers different advantages and limitations:

- The [Pixel Streaming](https://docs.unrealengine.com/en-us/Platforms/PixelStreaming) feature introduced in Unreal Engine 4.21.0 is currently Windows-only, **and is therefore not compatible with container-based cloud rendering.**

- Traditional Linux video capture mechanisms can be used inside containers, but typically require additional configuration and sometimes even additional container security permissions. These techniques also scale poorly to large numbers of containers performing cloud rendering within a single host system, **and are not recommended.**

- The community-maintained [UE4Capture](https://github.com/adamrehn/UE4Capture) plugin is quite similar to the official Pixel Streaming functionality, albeit with full cross-platform support. The [ue4-docker project](../obtaining-images/ue4-docker) includes UE4Capture in the [ue4-full container image](https://adamrehn.com/docs/ue4-docker/building-images/available-container-images#ue4-full). It is worth noting that UE4Capture does not currently feature support for relaying user input, and is limited to capturing video and audio output from the Unreal Engine only.

- The framebuffer can be captured programmatically from within the Unreal Engine itself, either via the [HighResShot console command](https://docs.unrealengine.com/en-us/Engine/Basics/Screenshots) or the classes from the [MovieSceneCapture module](https://api.unrealengine.com/INT/API/Runtime/MovieSceneCapture/index.html). Audio output can be similarly captured through the AudioMixer system by registering a submix buffer listener that implements the [ISubmixBufferListener interface](https://api.unrealengine.com/INT/API/Runtime/Engine/ISubmixBufferListener/index.html) that was introduced in Unreal Engine 4.20.0.


## Related media

{% include related-items/media.html url=page.url %}


## Related repositories

{% include related-items/repositories.html url=page.url %}
