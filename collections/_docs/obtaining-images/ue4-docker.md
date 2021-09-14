---
title: ue4-docker
tagline: Build Unreal Engine development container images using the ue4-docker project.
source: ue4-docker
order: 4
---

{% include alerts/source-features.html source=page.source %}


## About this container image source

Created in 2018, the ue4-docker project was the first open source project to provide Dockerfiles for building both Windows and Linux containers that include the Unreal Engine Build Tools. Since its initial creation, the project has grown to include support for building images compatible with the [NVIDIA Container Toolkit](../concepts/nvidia-docker), as well as convenience features such as automated configuration of Linux and Windows Server host systems.

The ue4-docker project is maintained by [Dr Adam Rehn](https://adamrehn.com/), the founder of both the [Unreal Engine For Research](https://ue4research.org/) initiative and the [Unreal Containers community hub]({{ "/" | relative_url }}). As a pioneering source of Dockerfiles for the Unreal Engine, the ue4-docker project has played a pivotal role in demonstrating the feasibility of Unreal Engine containers, and has been the initial point of discovery for many developers who now utilise Unreal Engine containers as part of their daily workflows.

Extensive documentation is provided for ue4-docker, including detailed guides on configuring host systems, customising and troubleshooting the image build process, and using convenience features such as automatic host configuration and container components export. The structure of the documentation hosted on the Unreal Containers community hub is directly based on the structure of the ue4-docker documentation, so navigating the available topics should feel simple and familiar.


## Using this container image source

- Start by exploring the project's [extensive documentation](https://docs.adamrehn.com/ue4-docker/), which provides everything needed to start building container images with ue4-docker.
- Check out the project's [GitHub repository](https://github.com/adamrehn/ue4-docker) to browse the contents of the underlying Dockerfiles.
- There are a series of [related articles](https://adamrehn.com/articles/tag/Unreal%20Engine/) on the author's website that provide background detail on the process of creating the initial version of the Dockerfiles and discuss the challenges that needed to be overcome.

The [Code section]({{ "/code" | relative_url }}) of the Unreal Containers community hub also features a number of example code repositories that work with container images built by ue4-docker.
