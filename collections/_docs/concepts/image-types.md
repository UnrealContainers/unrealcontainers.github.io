---
title: Development images vs. runtime images
tagline: What are the two types of Unreal Engine container images and how do they differ?
order: 1
---

{% capture _alert_content %}
- **Development images are used for tasks that require the Editor, such as building and packaging projects or plugins**. They contain the Unreal Engine build tools and Editor, and there are strict EULA restrictions that govern their distribution.

- **Runtime images are used for running packaged projects**. They contain only packaged Unreal projects and their runtime dependencies, and can be distributed more freely than development images.

- Different use cases require may either one or both of these types of container images.

{% endcapture %}
{% include alerts/overview.html content=_alert_content %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Container image types

When discussing container images in general, it is helpful to distinguish between two distinct types of images: **development images** and **runtime images**. The fundamental difference between these image types is that development images are used for **building software**, whereas runtime images are used for **running built software**. This distinction is quite common when dealing with compiled programming languages, since the tools required to build software are often quite large, but it is desirable that container images for running built software be as small as possible. The Docker [multi-stage build feature](https://docs.docker.com/develop/develop-images/multistage-build/) was created to make it easy for developers to use larger development container images to build their software and then copy the built files into a smaller runtime container image for deployment.

Here are some examples of programming languages and frameworks that provide separate development and runtime container images:

- **Microsoft .NET:** Microsoft provides the [.NET SDK](https://hub.docker.com/_/microsoft-dotnet-sdk/) development image for building .NET applications and the [.NET Runtime](https://hub.docker.com/_/microsoft-dotnet-runtime/) runtime image for running built .NET applications.

- **Java:** Oracle provides the [Oracle JDK](https://hub.docker.com/_/oracle-jdk) development image for building Java applications and the [Oracle Java 8 SE (Server JRE)](https://hub.docker.com/_/oracle-serverjre-8) runtime image for running built Java applications.

- **Go:** Google provides the [Go](https://hub.docker.com/_/golang) development image for building Go applications and the [distroless](https://github.com/GoogleContainerTools/distroless) runtime image for running built Go applications. (It is worth noting that the distroless runtime image actually includes variants for a number of different programming languages and Go is just one supported language.)

- **NVIDIA CUDA:** NVIDIA provides a ["devel" variant](https://gitlab.com/nvidia/container-images/cuda/-/tree/master/dist/11.3.0/ubuntu20.04-x86_64/devel) of the [NVIDIA CUDA](https://hub.docker.com/r/nvidia/cuda) container image for building CUDA applications and a ["runtime" variant](https://gitlab.com/nvidia/container-images/cuda/-/tree/master/dist/11.3.0/ubuntu20.04-x86_64/runtime) for running built CUDA applications.

Is important to note that **using one container image type does not force you to use the other type.** Development images can be used to build software that will run outside of a container, and runtime images can be used to run software that has been built outside of a container. Although it is often convenient to use development and runtime container images together (typically in the form of a Docker multi-stage build), it is never mandatory.


## Unreal Engine container image types

Unreal Engine container images are divided into development images and runtime images in much the same manner as other frameworks for compiled programming languages. Unreal Engine development images contain the Engine build tools and Editor, whereas Unreal Engine runtime images contain only packaged Unreal projects and their runtime dependencies. The nature of the Unreal Engine actually makes this distinction even more important than it is for other programming frameworks, for two key reasons:

- The Unreal Engine build tools and Editor are many gigabytes in size, which makes Unreal Engine development images far too large to use for deploying packaged Unreal projects.

- The Unreal Engine EULA imposes [strict limitations on the distribution of the Engine build tools and Editor](../obtaining-images/eula-restrictions) that do not apply to packaged Unreal projects, which means there is a legal distinction between Unreal Engine development images and runtime images.

{% include alerts/warning.html content="**It is critical that all Unreal Engine licensees who are using Unreal Engine containers are aware of the legal differences between development images and runtime images.** For this reason, it is strongly recommended that you ensure you have read and understood the entirety of the [Unreal Engine EULA Restrictions](../obtaining-images/eula-restrictions) page." %}


## Development images

{% include alerts/info.html content="Interested in building or obtaining a development image? Take a look at the [available sources of development images](../obtaining-images/image-sources#sources-of-unreal-engine-development-images)." %}

Unreal Engine development images are used for tasks that require the Unreal Editor, such as building and packaging Unreal projects or plugins, invoking command-line features such as commandlets or cinematic renders, and running automation workflows. They contain the following:

- ***(Required):*** The Unreal Engine build tools and Editor *(the Unreal Engine EULA refers to these as the **"Engine Tools"**)*
- ***(Optional):*** Non-core Engine components such as template projects, debug symbols and [Derived Data Cache (DDC)](https://docs.unrealengine.com/en-US/ProductionPipelines/DerivedDataCache/index.html)
- ***(Optional):*** Runtime dependencies and configuration files needed to enable the Unreal Editor to render graphics
- ***(Optional):*** Runtime dependencies and configuration files needed to [run the Unreal Editor as a sandboxed GUI application](../use-cases/linux-sandboxed-editor) under Linux
- ***(Optional):*** Runtime dependencies and configuration files needed to access the Unreal Editor remotely over VNC or SSH with X11 forwarding
- ***(Optional):*** Additional third-party extras or open source tools such as [ue4cli](https://github.com/adamrehn/ue4cli)

You **need to use a development image** if you want to perform any of the following tasks:

{% capture _list %}
- Build and package Unreal projects or plugins inside a container
- Run [automation tests](https://docs.unrealengine.com/en-US/TestingAndOptimization/Automation/index.html) for an Unreal project inside a container
- Run commandlets to bake lighting, cook content or perform other tasks for an Unreal project inside a container
- Render [Sequencer](https://docs.unrealengine.com/4.26/en-US/AnimatingObjects/Sequencer/Overview/) cinematics for an Unreal project inside a container
- Run [Python scripts or other Editor automation utilities](https://docs.unrealengine.com/en-US/ProductionPipelines/ScriptingAndAutomation/index.html) inside a container
{% endcapture %}
{% include styled-lists/checks.html list=_list %}

You **do not need to use a development image** if you want to do any of the following:

{% capture _list %}
- Build and package an Unreal project outside of a container and then run it inside a container using an Unreal Engine runtime image
{% endcapture %}
{% include styled-lists/crosses.html list=_list %}


## Runtime images

{% include alerts/info.html content="Interested in building or obtaining a runtime image? Take a look at the [available sources of runtime images](../obtaining-images/image-sources#sources-of-unreal-engine-runtime-images)." %}

Unreal Engine runtime images are used for running packaged Unreal projects. These images are typically **base images**, which means they are designed to act as a foundation that is then extended by developers who build other container images atop them. (In the case of Unreal Engine runtime images specifically, this extension takes the form of providing the files for the packaged Unreal project that you want to run in a container.) They contain the following:

- ***(Required):*** The runtime dependencies needed in order to run packaged Unreal projects with offscreen rendering
- ***(Optional):*** Runtime dependencies and configuration files needed to enable audio output
- ***(Optional):*** Runtime dependencies and configuration files needed to access packaged Unreal projects remotely using [Pixel Streaming](https://docs.unrealengine.com/en-US/SharingAndReleasing/PixelStreaming/index.html)
- ***(Optional):*** Runtime dependencies and configuration files needed to access packaged Unreal projects remotely over VNC or SSH with X11 forwarding
- ***(You add this yourself by either extending the base image or [bind-mounting files](https://docs.docker.com/storage/bind-mounts/))***: the packaged Unreal project that you want to run in a container

You **need to use a runtime image** if you want to perform any of the following tasks:

{% capture _list %}
- Build and package an Unreal project inside a container using an Unreal Engine development image and then run it inside a container
- Build and package an Unreal project outside of a container and then run it inside a container
{% endcapture %}
{% include styled-lists/checks.html list=_list %}

You **do not need to use a runtime image** if you want to do any of the following:

{% capture _list %}
- Build and package an Unreal project inside a container using an Unreal Engine development image and then run it outside of a container
{% endcapture %}
{% include styled-lists/crosses.html list=_list %}
