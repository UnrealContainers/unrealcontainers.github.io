---
title: Pixel Streaming
tagline: Render video and stream it to web browsers in real-time over WebRTC for interactive use.
quickstart: "run"
order: 2
---

{% capture _alert_content %}
- An [Unreal Engine runtime image](../concepts/image-types) with support for both [audio output](../concepts/audio-output) and [GPU acceleration](../concepts/gpu-acceleration), the latter of which must **include access to video encoding APIs**

- An environment [configured for running containers](../environments) with GPU acceleration
{% endcapture %}
{% include alerts/required.html content=_alert_content %}

{% include alerts/warning.html title="Changes incoming!" content="The contents of this page will be revised significantly once Unreal Engine 4.27 is released. The information presented here is currently accurate as at the time of writing." %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Overview and history

The Unreal Engine's [Pixel Streaming](https://docs.unrealengine.com/en-US/Platforms/PixelStreaming/index.html) system provides the ability to stream audio and video output from an Unreal application to client devices such as web browsers in real-time using [WebRTC](https://webrtc.org/) communication stack and to receive control data such as keyboard and mouse/touch inputs back from client devices. As discussed in the official Epic Games white paper [Streaming Unreal Engine content to multiple platforms](https://www.unrealengine.com/en-US/tech-blog/discover-pixel-streaming-real-time-distributed-content-for-any-device), Pixel Streaming allows developers to leverage cloud computing power to deliver high-quality experiences to the broadest possible range of hardware devices.

The reference implementation of the Pixel Streaming system was [introduced in Unreal Engine version 4.21](https://docs.unrealengine.com/4.26/en-US/WhatsNew/Builds/ReleaseNotes/4_21/#new:pixelstreaming_beta_). Initially, the reference implementation supported only the Windows operating system. This precluded the use of Pixel Streaming inside Unreal Engine containers, since [GPU accelerated Windows containers](../concepts/gpu-acceleration#gpu-support-for-windows-containers) do not have access to the required video encoding APIs. Although newer versions of the Unreal Engine have subsequently introduced Linux support in order to facilitate use inside containers, it is important to note that the limitations of Windows containers still make them unsuitable for use with Pixel Streaming. **Pixel Streaming cannot be used in GPU accelerated Windows containers, irrespective of the version.**

Developers from the company [TensorWorks](../../support#tensorworks) added support for both Linux and [GPU accelerated Linux containers](../concepts/gpu-acceleration) to the Pixel Streaming system in custom versions of both [Unreal Engine 4.23](https://github.com/adamrehn/UnrealEngine/tree/4.23.1-pixelstreaming) and [Unreal Engine 4.25](https://github.com/ImmortalEmperor/UnrealEngine/tree/4.25-pixelstreaming). These custom forks of the Unreal Engine are collectively referred to as ***"Pixel Streaming for Linux"*** in order to differentiate them from the official reference implementation. Community interest in container-based Pixel Streaming eventually prompted Epic Games to [officially announce Linux and container support for Pixel Streaming on their public roadmap](https://portal.productboard.com/epicgames/1-unreal-engine-public-roadmap/c/319-pixel-streaming-now-production-ready) for inclusion in the release of Unreal Engine 4.27.

**Until the official release of Unreal Engine 4.27, the [TensorWorks fork of Unreal Engine 4.25](https://github.com/ImmortalEmperor/UnrealEngine/tree/4.25-pixelstreaming) remains the recommended version for developers seeking to use Pixel Streaming in containers.**


## Key considerations

- **Note: most of these recommendations will change as soon as Unreal Engine 4.27 is released.**

- The 4.25 version of Pixel Streaming for Linux only supports NVIDIA GPUs, so you will need to use the [NVIDIA Container Toolkit](../concepts/nvidia-docker) to run Pixel Streaming applications in GPU accelerated containers.

- The 4.25 version of Pixel Streaming for Linux uses the NVIDIA CUDA API, so you will need a runtime image that supports both [audio output](../concepts/audio-output) and CUDA. The recommended container image is currently [adamrehn/ue4-runtime:18.04-cudagl11.1.1](https://hub.docker.com/layers/adamrehn/ue4-runtime/18.04-cudagl11.1.1-virtualgl/images/sha256-2eca157787a34c5c9cf56649f939615c6c5626e0915a4535ab6b378c2333e182?context=explore). Note that this base image does not have NVIDIA NVENC support enabled by default, so you will need to extend it by creating your own container image and including the following directive in your Dockerfile:
    
    ```dockerfile
# Enable the NVIDIA driver capabilities required by the NVENC video encoding API
ENV NVIDIA_DRIVER_CAPABILITIES ${NVIDIA_DRIVER_CAPABILITIES},video
```

- The networking stacks provided by container runtimes such as Docker introduce additional overheads that result in increased latency of UDP packets and thus reduced responsiveness of WebRTC streams. It is strongly recommended that you run containers with [host networking mode](https://docs.docker.com/network/host/) enabled in order to ensure the smoothest experience for users. *(This recommendation will remain the same even after Unreal Engine 4.27 is released.)*


## Implementation guidelines

### Basic setup

You will need two containers for a minimal deployment of Pixel Streaming:

- **Pixel Streaming application:** this container encapsulates the packaged binaries for your Unreal project. The project must have the Pixel Streaming plugin enabled and the binaries must be packaged using [Pixel Streaming for Linux, version 4.25](https://github.com/ImmortalEmperor/UnrealEngine/tree/4.25-pixelstreaming). **The container must run with GPU acceleration enabled and should ideally be run with host networking mode enabled.**
    
    The container's entrypoint should be configured to run the packaged project on start up, like so:
    
    ```dockerfile
# Set the project as the container's entrypoint
# (Replace "/home/ue4/project/MyProject.sh" with the path to your project's startup script)
# 
# Note that we use 127.0.0.1 as the IP address for Cirrus here,
# which only works if both containers are running on the same host system
# 
ENTRYPOINT ["/home/ue4/project/MyProject.sh", "-RenderOffscreen", "-Windowed", "-ForceRes", "-ResX=1920", "-ResY=1080", "-PixelStreamingIP=127.0.0.1", "-PixelStreamingPort=8888"]
```

- **Cirrus signalling server:** this container encapsulates Cirrus, which acts as the WebRTC signalling server for the Pixel Streaming application and also serves the static client files (HTML, CSS, Javascript, etc.) to web browsers. You can find an [example Dockerfile for Cirrus](https://github.com/adamrehn/ue4-example-dockerfiles/blob/master/pixel-streaming/4.23/server/Dockerfile) in the [ue4-example-dockerfiles](https://github.com/adamrehn/ue4-example-dockerfiles) repository on GitHub. (Note that the example Dockerfile is designed to copy the source code for Cirrus from a [development image](../concepts/image-types#development-images) for the 4.23 version of Pixel Streaming for Linux, but you can modify this to copy the files from either a 4.25 development image or from the host filesystem.) **The container must be run with host networking mode enabled if the container for the Pixel Streaming application has host networking mode enabled.**

Once you have started both of these containers together, you should be able to access your Pixel Streaming application by opening a web browser and navigating to <http://127.0.0.1> (or the public IP address of the machine that is running the containers if you are accessing the Pixel Streaming application from a different machine.)

### Configuring STUN and TURN

You can use the `peerConnectionOptions` [configuration parameter for Cirrus](https://docs.unrealengine.com/4.26/en-US/SharingAndReleasing/PixelStreaming/PixelStreamingReference/#signalingserverconfigurationparameters) to specify the STUN and TURN servers that should be used when establishing WebRTC connections between web browsers and your Pixel Streaming application. This makes it possible to access Pixel Streaming applications from all client devices even when firewalls or intermediate network topology would otherwise prevent the establishment of direct peer-to-peer connections. There are a variety of options available, but the following are recommended:

- **STUN:** you can use any STUN server, including the free STUN servers provided by Google (e.g. `"stun:stun.l.google.com:19302"`.) Note that the STUN server should always be running on a machine in a different network to the machine running the Pixel Streaming application, since STUN relies on this separation to ensure the correct public IP address is reported.

- **TURN:** we recommend the use of the open source [coturn](https://github.com/coturn/coturn) server for providing TURN services. An [official container image is available](https://hub.docker.com/r/coturn/coturn) and unlike a STUN server, the TURN server can be run on the same machine as the Pixel Streaming application in order to minimise latency.

### Deploying at scale

The Cirrus signalling server and the Matchmaker Server that ship with the reference implementation of Pixel Streaming are only designed to act as a starting point for developers to extend, and do not provide the necessary functionality for deploying Pixel Streaming applications at scale. TensorWorks is currently developing the open source [Scalable Pixel Streaming Framework](https://scalablestreaming.io) to address this. This section will be updated when the framework is made available to the public.


## Related reading

- [Adam Rehn: Pixel Streaming in Linux containers](https://adamrehn.com/articles/pixel-streaming-in-linux-containers/). This is the original announcement blog post for Pixel Streaming for Linux that provided details on how to run both the 4.23 and 4.25 versions in GPU accelerated Linux containers.

- [Epic Games: Pixel Streaming Hosting and Networking Guide](https://docs.unrealengine.com/4.26/en-US/SharingAndReleasing/PixelStreaming/Hosting/). This page from the official Unreal Engine documentation provides an overview of how the various components of the Pixel Streaming system are designed to interact when deployed in the cloud.


## Related media

{% include related-items/media.html url=page.url %}


## Related repositories

{% include related-items/repositories.html url=page.url %}
