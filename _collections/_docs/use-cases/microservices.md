---
title: Microservices
tagline: Build and deploy cloud native microservices powered by Unreal Engine technology.
quickstart: ["build", "run"]
order: 3
---

{% capture _alert_content %}
- Base container image(s) that [support running packaged Unreal projects](../obtaining-images/image-sources) (the images will need NVIDIA Docker support if the microservice performs rendering)
- An environment [configured for running containers](../environments) (a Linux environment with NVIDIA Docker if performing rendering)
{% endcapture %}
{% include alerts/required.html content=_alert_content %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Overview

Containerised microservices are an extremely popular architectural paradigm for implementing server-side applications. Unreal Engine containers allow this same architecture to be applied to microservices powered by the Unreal Engine. Infrastructure for integrating existing RPC frameworks allows developers to implement Unreal microservices using familiar technologies, while the [NVIDIA Docker runtime](../concepts/nvidia-docker) allows Unreal microservices to perform 2D or 3D rendering with full GPU acceleration. Compatibility with container orchestration technologies means developers can deploy and scale Unreal microservices in exactly the same manner as traditional microservices.


## Key considerations

- This page only discusses topics that are specifically relevant to Unreal microservices. **If you are creating microservices that perform rendering then be sure to also read the [cloud rendering](./cloud-rendering) page and familiarise yourself with the relevant details.**

- Because NVIDIA Docker only works with Linux containers running under Linux host systems, microservices that perform rendering cannot run inside [Windows containers](../concepts/windows-containers). Although Windows containers can be used to run microservices that do not perform any rendering, Linux containers are still strongly recommended due to better compatibility with container orchestration technologies and easier installation of runtime dependencies using system package managers.

- Any third-party C++ libraries that will be integrated with the Unreal Engine under Linux must be built against libc++ instead of libstdc++, since the Unreal Engine itself is built against its own bundled copy of libc++. Third-party libraries that rely on dependencies which are also bundled with the Unreal Engine may also encounter issues related to symbol interposition. See [this article discussing the creation of the conan-ue4cli project](https://adamrehn.com/articles/cross-platform-library-integration-in-unreal-engine-4/) for a detailed explanation of these issues.


## Implementation guidelines

### Integrating an RPC framework

{% include alerts/info.html content="Note that the [Web Remote Control](https://docs.unrealengine.com/en-US/Engine/Editor/ScriptingAndAutomation/WebControl/index.html) feature introduced in Unreal Engine 4.23 only works in the Unreal Editor, not in packaged projects. This means it cannot be used as an RPC framework for deployed microservices." %}

#### Integrating RPC frameworks natively in C++

The most performant option for integrating native RPC frameworks with the Unreal Engine is to do so in C++. However, there are a number of complexities that must be addressed in order to correctly integrate third-party C++ libraries with the Unreal Engine under Linux which make manual compilation or the use of prebuilt system libraries difficult or impossible. We strongly recommend making use of infrastructure such as the [conan-ue4cli project](https://github.com/adamrehn/conan-ue4cli) to build any third-party C++ libraries in a manner that will ensure the resulting binaries are compatible with the Unreal Engine.

Existing solutions are available for building Unreal-compatible binaries for the following native RPC frameworks:

- [gRPC](https://grpc.io/): developed by Google, this framework is extremely popular and has been adopted as a project of the [Cloud Native Computing Foundation](https://www.cncf.io/). gRPC provides bindings in a wide variety of languages, including C++. A recipe for building an Unreal-compatible version of the gRPC C++ bindings is [bundled with the conan-ue4cli project](https://github.com/adamrehn/ue4-conan-recipes).

#### Integrating RPC frameworks using other language integrations

RPC frameworks for non-native programming languages can be integrated by making use of community-maintained projects that add support for these programming languages in the Unreal Engine. Although additional complexity is introduced by the integration of another programming language, RPC frameworks written for these languages do not require special compilation to work with the Unreal Engine.

There are a number of community-maintained projects that integrate non-native programming languages with the Unreal Engine. The following list comes from the [Unreal Engine For Research](https://ue4research.org/resources#integrations) initiative:

- [UnrealEnginePython](https://github.com/20tab/UnrealEnginePython): this plugin adds runtime support for the Python programming language and provides comprehensive bindings for the Unreal Engine API. Code running through this plugin can make use of any RPC framework with Python bindings.

- [Unreal.js](https://github.com/ncsoft/Unreal.js): this plugin adds runtime support for the Javascript programming language and provides comprehensive bindings for the Unreal Engine API. Code running through this plugin can make use of any RPC framework with Javascript bindings.

- [MonoUE](https://mono-ue.github.io/): this plugin adds runtime support for the C# and F# programming languages and provides bindings for the Unreal Engine API. Code running through this plugin can make use of any RPC framework with .NET bindings. Note that this project is still under development and there are a number of features that are yet to be implemented.

### Building container images for deployment

See the [relevant sections from the cloud rendering page](./cloud-rendering#approaches) for details on selecting an appropriate base image and building your own container images for deployment.


## Related media

{% include related-items/media.html url=page.url %}


## Related repositories

{% include related-items/repositories.html url=page.url %}
