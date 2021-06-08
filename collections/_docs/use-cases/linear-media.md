---
title: Rendering Linear Media
tagline: Perform batch rendering of images or video for non-interactive use.
quickstart: "workflows"
order: 6
---

{% capture _alert_content %}
- An [Unreal Engine development image](../concepts/image-types) with support for [GPU acceleration](../concepts/gpu-acceleration) or [software rendering](../concepts/gpu-acceleration#software-rendering)
- An environment [configured for running containers](../environments) with GPU acceleration (unless using software rendering)
{% endcapture %}
{% include alerts/required.html content=_alert_content %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Overview

Rendering images and video with the Unreal Engine for non-interactive use is becoming increasingly popular in industries such as automotive, architectural visualisation, and film and television. Unreal Engine containers make it possible to perform these rendering tasks at scale in the cloud.


## Key considerations

- You will need to start the Unreal Editor with the `-RenderOffscreen` command-line flag in order to perform offscreen rendering inside a container.

- The Unreal Engine does not currently support raytracing using Vulkan on Linux. If your use case requires real-time raytracing then you will need to use [GPU accelerated Windows containers](../concepts/gpu-acceleration#gpu-support-for-windows-containers).

- Rendering video frames with Sequencer using Vulkan on either Windows or Linux currently prints a series of non-fatal errors to the Engine's log output. These errors do not appear to interfere with the quality of the rendered output.

- If you are rendering individual images on demand and transmitting them to users in response to incoming requests then you might be interested in exposing this functionality directly via an [Unreal-powered microservice](./microservices).


## Implementation guidelines

### Rendering individual images

The framebuffer can be captured programmatically from within the Unreal Engine itself, either via the [HighResShot console command](https://docs.unrealengine.com/en-us/Engine/Basics/Screenshots) or the classes from the [MovieSceneCapture module](https://api.unrealengine.com/INT/API/Runtime/MovieSceneCapture/index.html). How you choose to trigger the image capture will depend on the specific details of your use case.

### Rendering video frames

Cinematic sequences created using the Unreal Engine's [Sequencer](https://docs.unrealengine.com/4.26/en-US/AnimatingObjects/Sequencer/Overview/) system can be rendered to a set of video frames from the command-line using the flags described in the [Command Line Arguments for Rendering Movies](https://docs.unrealengine.com/en-US/AnimatingObjects/Sequencer/Workflow/RenderAndExport/RenderingCmdLine/index.html) page of the official Unreal Engine documentation. These same flags also work inside Unreal Engine containers, albeit with the necessary addition of the `-RenderOffscreen` flag.

As an example, the following commands can be run from your Unreal project's root directory (the one containing the `.uproject` file) to render a video sequence inside a GPU accelerated Linux container:

{% highlight bash %}

# Replace this value with the asset reference for your map
RENDER_MAP='/Game/Maps/ArchVis_Lightmap'

# Replace this value with the asset reference for your sequencer suquence
RENDER_SEQUENCE='/Game/Cinematic/archviz_cine_MASTER_MaxQuality.archviz_cine_MASTER_MaxQuality'

# This will render out frames to a subdirectory called "Rendered" inside your Unreal project's root directory
# (Note that if you haven't built the DDC for your project then the Unreal Engine will compile all of the shaders for the map before it starts rendering frames)
docker run --rm -ti "-v`pwd`:/hostdir" -w /hostdir --gpus all 'adamrehn/ue4-full:4.26.2' \
    ue4 run \
    "$RENDER_MAP" \
    -MovieSceneCaptureType="/Script/MovieSceneCapture.AutomatedLevelSequenceCapture" \
    -LevelSequence="$RENDER_SEQUENCE" \
    -game -NoLoadingScreen -NoSplash -RenderOffscreen -ForceRes \
    -MovieFrameRate=30 -ResX=1280 -ResY=720 -MovieQuality=100 \
    -MovieFormat=PNG -NoTextureStreaming -MovieCinematicMode=yes -NoScreenMessages \
    -MovieFolder='/hostdir/Rendered'
{% endhighlight %}
