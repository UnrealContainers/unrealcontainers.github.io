---
title: Audio output in containers
tagline: How can audio output be enabled in Unreal Engine containers and when is it necessary?
order: 6
---

{% capture _alert_content %}
- Audio output must be enabled for use cases that leverage the Unreal Engine's audio mixing functionality.

- Multiple approaches are available for enabling audio output in Linux containers.

- No additional configuration is required for Windows containers.
{% endcapture %}
{% include alerts/overview.html content=_alert_content %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Audio output requirements

When running applications inside a container, there are no physical audio devices available by default. This is important when discussing Unreal Engine containers because the Engine's [Audio Mixer](https://docs.unrealengine.com/4.26/en-US/WorkingWithMedia/Audio/AudioMixer/) system must still be initialised in order to use audio mixing functionality, even when the generated audio data will be written to file or transmitted over a network rather than sent to an output device. This is particularly relevant for use cases such as [Pixel Streaming](https://docs.unrealengine.com/4.26/en-US/SharingAndReleasing/PixelStreaming/), which leverages the Audio Mixer system in order to capture generated audio and transmit it to client devices over WebRTC.

The Audio Mixer system supports the concept of a ["null audio device"](https://github.com/EpicGames/UnrealEngine/blob/4.26.2-release/Engine/Source/Runtime/AudioMixerCore/Public/AudioMixer.h#L592), which allows the Engine to generate audio output even when there are no physical audio devices available. However, support for using a null audio device varies between the backend implementations for different platforms. For more details, see the specific sections for [Linux](#audio-output-in-linux-containers) and [Windows](#audio-output-in-windows-containers).


## Audio output in Linux containers

The SDL2 audio backend for Linux **does not support the use of a null audio device.** As a result, the Audio Mixer system will only initialise successfully under Linux if one of the supported SDL audio drivers is available. These drivers include the [Advanced Linux Sound Architecture (ALSA)](https://alsa-project.org/) and [PulseAudio](https://www.freedesktop.org/wiki/Software/PulseAudio/):

- **ALSA** is the lower-level option, and provides less flexibility than a sound server such as PulseAudio. Creating an [ALSA loopback device](https://www.alsa-project.org/wiki/Matrix:Module-aloop) to act as a virtual audio output device involves loading kernel modules, which requires running containers with specific security privileges.

- **PulseAudio** is a higher-level sound server and provides far more flexibility as a result of its client/server model. Enabling a [PulseAudio null sink](https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/Modules/#module-always-sink) to act as a virtual audio output device requires only a single directive in a configuration file.

Due to its flexibility, PulseAudio is the recommended audio driver for Linux containers. There are multiple approaches available when configuring PulseAudio to suit the requirements of a given use case.

### Automatically spawning a PulseAudio server inside the container

The PulseAudio client can [automatically spawn a PulseAudio server on demand](https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/Running/#autospawning) if no existing server is available. (This is actually the default behaviour unless explicitly disabled in a configuration file.) So long as the PulseAudio server is configured to enable a null sink, this will ensure audio output is always available without the need to run a PulseAudio server manually or interact with the host system. **This is the recommended approach when running Unreal Engine containers on virtual machines in the cloud.**

If you are using [an existing runtime or development container image](../obtaining-images/image-sources) then be sure to check whether this functionality is already configured for you. If it is not configured or you are writing your own Dockerfiles then you can configure it yourself by specifying the appropriate directives in the PulseAudio configuration files.

First, ensure the following directive is present in the `/etc/pulse/default.pa` configuration file:

{% highlight bash %}
# Automatically create a null sink to act as a virtual audio output device
# when there are no physical audio devices available
load-module module-always-sink
{% endhighlight %}

By default, the null sink will use different audio parameters to those used by the Unreal Engine under Linux, which may lead to audio distortion issues. To prevent such issues, it is recommended that the following directives be added to the `/etc/pulse/daemon.conf` configuration file to ensure the default audio parameters match those used by the Unreal Engine:

{% highlight bash %}
# Use six audio channels at 48kHz with 32-bit little-endian floating point samples
default-sample-format = float32le
default-sample-rate = 48000
default-sample-channels = 6
{% endhighlight %}

### Using the host system's PulseAudio server

If you are running a PulseAudio server on the host system and want to route the Engine's audio output to a physical audio device (such as a speaker) then you can configure the container's PulseAudio client to connect to the host system's server. **This is primarily useful when running Unreal Engine containers on local machines rather than in the cloud.**

If you are using [an existing runtime or development container image](../obtaining-images/image-sources) then be sure to check whether this functionality is already configured for you. If it is not configured or you are writing your own Dockerfiles then you can configure it yourself by specifying the following entries in the `/etc/pulse/client.conf` configuration file:

{% highlight bash %}
# Connect to the host system's PulseAudio server using the bind-mounted UNIX socket
default-server = unix:/run/user/1000/pulse/native

# Prevent a PulseAudio server from attempting to spawn in the container
autospawn = no
daemon-binary = /bin/true

# Prevent the use of shared memory when communicating with the PulseAudio server
enable-shm = false
{% endhighlight %}

Once this is configured, you will need to bind-mount the PulseAudio socket from the host system when running your containers. You can do so by specifying the flag `"-v/run/user/$UID/pulse:/run/user/1000/pulse"` when invoking the [docker run](https://docs.docker.com/engine/reference/run/) command.


## Audio output in Windows containers

The XAudio2 audio backend for Windows **supports the use of a null audio device** (and was in fact the first platform for which support was implemented.) As a result, the Audio Mixer system will initialise successfully under Windows even when there are no physical audio devices available. This means that no additional configuration is required to enable audio output in a Windows container.
