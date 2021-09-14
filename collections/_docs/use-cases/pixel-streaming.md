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


## Contents
{:.no_toc}

* TOC
{:toc}


## Overview and history

The Unreal Engine's [Pixel Streaming](https://docs.unrealengine.com/en-US/Platforms/PixelStreaming/index.html) system provides the ability to stream audio and video output from an Unreal application to client devices such as web browsers in real-time using [WebRTC](https://webrtc.org/) communication stack and to receive control data such as keyboard and mouse/touch inputs back from client devices. As discussed in the official Epic Games white paper [Streaming Unreal Engine content to multiple platforms](https://www.unrealengine.com/en-US/tech-blog/discover-pixel-streaming-real-time-distributed-content-for-any-device), Pixel Streaming allows developers to leverage cloud computing power to deliver high-quality experiences to the broadest possible range of hardware devices.

The reference implementation of the Pixel Streaming system was [introduced in Unreal Engine version 4.21](https://docs.unrealengine.com/4.26/en-US/WhatsNew/Builds/ReleaseNotes/4_21/#new:pixelstreaming_beta_). Initially, the reference implementation supported only the Windows operating system. Developers from the company [TensorWorks](../../support#tensorworks) added support for both Linux and [GPU accelerated Linux containers](../concepts/gpu-acceleration) to the Pixel Streaming system in custom versions of both [Unreal Engine 4.23](https://github.com/adamrehn/UnrealEngine/tree/4.23.1-pixelstreaming) and [Unreal Engine 4.25](https://github.com/ImmortalEmperor/UnrealEngine/tree/4.25-pixelstreaming). These custom forks of the Unreal Engine were collectively referred to as "[Pixel Streaming for Linux](https://adamrehn.com/articles/pixel-streaming-in-linux-containers/)" in order to differentiate them from the official reference implementation.

The features from the Pixel Streaming for Linux project were merged into the upstream reference implementation [in Unreal Engine 4.27](https://docs.unrealengine.com/4.27/en-US/WhatsNew/Builds/ReleaseNotes/4_27/#pixelstreaming), alongside official support for containers. **The Pixel Streaming for Linux project is now deprecated and the reference implementation in Unreal Engine 4.27 or newer is the recommended version for developers seeking to use Pixel Streaming in containers.**


## Key considerations

- The networking stacks provided by container runtimes such as Docker introduce additional overheads that result in increased latency of UDP packets and thus reduced responsiveness of WebRTC streams. It is strongly recommended that you run Linux containers with [host networking mode](https://docs.docker.com/network/host/) enabled in order to ensure the smoothest experience for users. **Windows containers do not support host networking mode and will always suffer from increased WebRTC latency.**

- The only way to run Pixel Streaming inside GPU accelerated Windows containers is by enabling graphics APIs for encoding video as described in the blog post [Enabling vendor-specific graphics APIs in Windows containers](../../blog/enabling-vendor-specific-graphics-apis-in-windows-containers/). **Using graphics APIs other than DirectX inside Windows containers is not officially supported by Microsoft and is not recommended for production use**.

- As part of the [official container support](../obtaining-images/official-images) that was [introduced in Unreal Engine 4.27](https://docs.unrealengine.com/4.27/en-US/WhatsNew/Builds/ReleaseNotes/4_27/#containerdeployment_beta_), Epic Games provides both Linux and Windows runtime images for Pixel Streaming applications:
    
    - `ghcr.io/epicgames/unreal-engine:runtime-pixel-streaming`: this is the official Pixel Streaming runtime image for GPU accelerated Linux containers. It extends the [adamrehn/ue4-runtime:18.04-cudagl11.1.1](https://hub.docker.com/r/adamrehn/ue4-runtime) base image with NVIDIA NVENC support. You will need to use the [NVIDIA Container Toolkit](../concepts/nvidia-docker) to run containers based on this image. (Support for AMD GPUs is [planned for a future release](https://docs.unrealengine.com/4.27/en-US/SharingAndReleasing/Containers/KnownLimitations/).)
    
    - `ghcr.io/epicgames/unreal-engine:runtime-windows`: this is the official runtime image for GPU accelerated Windows containers. It includes the code to enable additional graphics APIs from the blog post [Enabling vendor-specific graphics APIs in Windows containers](../../blog/enabling-vendor-specific-graphics-apis-in-windows-containers/).

- **It is strongly recommended that you use GPU accelerated Linux containers to deploy Pixel Streaming applications.** Deployment of Pixel Streaming applications in GPU accelerated Windows containers has not been tested thoroughly and is recommended for experimental use only.


## Implementation guidelines

### Example configuration

Unreal Engine version 4.27.0 and newer ships with a [Docker Compose](https://docs.docker.com/compose/) configuration which demonstrates a minimal deployment of a Pixel Streaming application using Linux containers. The example files can be found under the [Engine/Extras/Containers/Examples/PixelStreaming](https://github.com/EpicGames/UnrealEngine/tree/release/Engine/Extras/Containers/Examples/PixelStreaming) directory of the Unreal Engine source tree. **Note that if you have downloaded the Unreal Engine source code from GitHub then you will need to run `Setup.bat` (under Windows) or `Setup.sh` (under other platforms) to retrieve the source files for the example, since these files are currently treated as binary dependencies and are not included in the git repository itself.**

The example configuration includes the following components, each of which is discussed in the sections that follow:

- A demo Pixel Streaming application
- The Cirrus signalling server
- The [coturn](https://github.com/coturn/coturn) TURN server

### Basic setup

You will need two containers for a minimal deployment of Pixel Streaming:

- **Pixel Streaming application:** this container encapsulates the packaged binaries for your Unreal project. The project must have the Pixel Streaming plugin enabled and the binaries must be packaged using Unreal Engine 4.27.0 or newer. The container image should extend the official Pixel Streaming runtime image for your platform as listed in the [Key considerations](#key-considerations) section above. The container must run with GPU acceleration enabled and under Linux should ideally be run with host networking mode enabled.
    
    The container's entrypoint should be configured to run the packaged project on startup, like so:
    
    ```dockerfile
# Set the project as the container's entrypoint
# (Replace "/home/ue4/project/MyProject.sh" with the path to your project's startup script)
# 
# Note that we use 127.0.0.1 as the IP address for Cirrus here,
# which only works if both containers are running on the same host system in host networking mode
# 
ENTRYPOINT ["/home/ue4/project/MyProject.sh", "-RenderOffscreen", "-Windowed", "-ForceRes", "-ResX=1920", "-ResY=1080", "-PixelStreamingIP=127.0.0.1", "-PixelStreamingPort=8888"]
```

- **Cirrus signalling server:** this container encapsulates Cirrus, which acts as the WebRTC signalling server for the Pixel Streaming application and also serves the static client files (HTML, CSS, Javascript, etc.) to web browsers. An official container image for Cirrus ships with Unreal Engine 4.27 with the name `ghcr.io/epicgames/pixel-streaming-signalling-server:4.27`. Note that this container will also need to run with host networking mode enabled if the container for the Pixel Streaming application has host networking mode enabled and is attempting to access Cirrus via the address `127.0.0.1`. (This is not required if the containers are running on separate machines and the ports for Cirrus are exposed via its host machine's IP address or via a load balancer or other mechanism.)

Once you have started both of these containers together, you should be able to access your Pixel Streaming application by opening a web browser and navigating to <http://127.0.0.1> (or the public IP address of the machine that is running the Cirrus container if you are accessing the Pixel Streaming application from a different machine.)

### Configuring STUN and TURN

You can use the `peerConnectionOptions` [configuration parameter for Cirrus](https://docs.unrealengine.com/4.27/en-US/SharingAndReleasing/PixelStreaming/PixelStreamingReference/#signalingserverconfigurationparameters) to specify the STUN and TURN servers that should be used when establishing WebRTC connections between web browsers and your Pixel Streaming application. This makes it possible to access Pixel Streaming applications from all client devices even when firewalls or intermediate network topology would otherwise prevent the establishment of direct peer-to-peer connections. There are a variety of options available, but the following are recommended:

- **STUN:** you can use any STUN server, including the free STUN servers provided by Google (e.g. `"stun:stun.l.google.com:19302"`.) Note that the STUN server should always be running on a machine in a different network to the machine running the Pixel Streaming application, since STUN relies on this separation to ensure the correct public IP address is reported.

- **TURN:** we recommend the use of the open source [coturn](https://github.com/coturn/coturn) server for providing TURN services. An [official container image is available](https://hub.docker.com/r/coturn/coturn) and unlike a STUN server, the TURN server can be run on the same machine as the Pixel Streaming application in order to minimise latency.

### Deploying at scale

The Cirrus signalling server and the Matchmaker Server that ship with the reference implementation of Pixel Streaming are only designed to act as a starting point for developers to extend, and do not provide the necessary functionality for deploying Pixel Streaming applications at scale. [TensorWorks](https://tensorworks.com.au) is currently developing the open source [Scalable Pixel Streaming Framework](https://scalablestreaming.io) to address this. This section will be updated when the framework is made available to the public.


## Related reading

- [Epic Games: Containers Overview](https://docs.unrealengine.com/4.27/en-US/SharingAndReleasing/Containers/ContainersOverview/). This page from the official Unreal Engine documentation provides an overview of container support in the Unreal Engine and lists the official container images that ship with Unreal Engine 4.27 and newer.

- [Epic Games: Pixel Streaming Hosting and Networking Guide](https://docs.unrealengine.com/4.27/en-US/SharingAndReleasing/PixelStreaming/Hosting/). This page from the official Unreal Engine documentation provides an overview of how the various components of the Pixel Streaming system are designed to interact when deployed in the cloud.


## Related media

{% include related-items/media.html url=page.url %}
