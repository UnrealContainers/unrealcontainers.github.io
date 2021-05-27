---
title: Linux Sandboxed Editor
tagline: Run the Unreal Editor inside a container and interact with it directly from the host system.
quickstart: "workflows"
order: 6
---

{% capture _alert_content %}
- Unreal Engine container image(s) that include both [the Engine Tools and NVIDIA Container Toolkit support](../obtaining-images/image-sources) as well as the X11 runtime libraries
- A [local Linux development environment](../environments/local-linux) configured for running containers via the NVIDIA Container Toolkit and running an X11 server
{% endcapture %}
{% include alerts/required.html content=_alert_content %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Overview

{% include alerts/warning.html title="Simpler alternative available:" content="This approach is far more restrictive than simply copying the Engine Installation to the host system and running it natively. In the majority of situations you will be better off following the instructions provided on the [Linux Installed Builds](./linux-installed-builds) page instead." %}

On Linux host systems that satisfy the requirements listed at the top of this page, it is possible to run the Unreal Editor from inside a Docker container with full GPU acceleration and have its UI windows displayed on the host system. This facilitates a deployment strategy where the Unreal Engine runs on developers' machines in a sandboxed environment that still allows users to interact with the Editor in the same manner as they would if it were running natively. This can be useful in scenarios where you need to bundle complex dependencies with the Editor (such as machine learning frameworks) or if you are using a custom Engine version that is updated very frequently and will benefit from the ability to rapidly deploy updates to developers' workstations.


## Key considerations

- This approach only works on machines that satisfy the [hardware and software requirements](https://github.com/NVIDIA/nvidia-docker/wiki/Installation-(Native-GPU-Support)) of the [NVIDIA Container Toolkit](../concepts/nvidia-docker). By contrast, exporting [Linux Installed Builds](./linux-installed-builds) and running them natively will work on any Linux machine that is capable of running the Unreal Editor, irrespective of GPU vendor.

- Because this approach makes use of a bind-mounted X11 socket, an X11 server must be running on the host system. Linux distributions that ship with an alternative display server protocol enabled by default (e.g. [Wayland](https://wayland.freedesktop.org/)) will require additional configuration to enable an X11 user session.

- Container images must include the X11 runtime libraries in order for the Unreal Engine to recognise the bind-mounted X11 socket. If your container images do not include the relevant libraries then the Editor will default to offscreen rendering and no UI windows will be displayed.


## Implementation guidelines

### Implementation-agnostic commands

#### Starting a container

Irrespective of the [source of container images](../obtaining-images/image-sources) that is used, the following command will start a container from which the Editor can be run interactively (replace `unreal-engine:latest` with the tag for your container image):

{% highlight bash %}
# Starts a container running a bash shell from which the UE4Editor executable can be run
docker run --rm -ti --gpus=all -v/tmp/.X11-unix:/tmp/.X11-unix:rw -e DISPLAY "unreal-engine:latest" bash
{% endhighlight %}

The individual arguments to the [docker run](https://docs.docker.com/engine/reference/run/) command are explained below:

- `--rm`: Removes the container when it stops.

- `-ti`: Starts an interactive session.

- `--gpus=all`: Enables GPU access via the NVIDIA Container Toolkit.

- `-v/tmp/.X11-unix:/tmp/.X11-unix:rw`: Bind-mounts the X11 socket from the host system.

- `-e DISPLAY`: Propagates the `DISPLAY` environment variable from the host system.

- `"unreal-engine:latest"`: **Replace this with the tag for your container image.**

- `bash`: Starts an interactive bash shell.

Once the interactive bash shell has started, you can run the `UE4Editor` executable using the appropriate path for your container image. If everything is working correctly, the Unreal Editor splash screen should be displayed on the host system's display. Once the Editor has completed loading then you can interact with it as you normally would if it were running on the host system.

#### Bind-mounting project directories

Any data stored in the container's filesystem will be erased when the container is removed. To create projects that will persist, or to access existing projects, you will need to [bind-mount](https://docs.docker.com/storage/bind-mounts/) the appropriate directories from the host filesystem.

#### Avoiding repeated shader recompilation

Under Linux, the Editor stores its cache for compiled assets under the directory `$HOME/.config/Epic`. By default, this directory exists only inside the container filesystem and is erased when the container is removed. To ensure the data in this directory persists between container runs, you can bind-mount this directory from the host system by adding the flag `"-v$HOME/.config/Epic:/home/ue4/.config/Epic"` to your [docker run](https://docs.docker.com/engine/reference/run/) command.

#### Enabling audio support

By default, the container will not have access to the audio devices from the host system and so audio output will be disabled. If your container images include PulseAudio support then you can enable audio output by bind-mounting the PulseAudio socket from the host system using the flag `"-v/run/user/$UID/pulse:/run/user/1000/pulse"`. Note that this will not work for the root user, so you will need to run the command as a non-root user as described by the [Post-installation steps for Linux](https://docs.docker.com/install/linux/linux-postinstall/) page of the Docker documentation.

### Using container images from the ue4-docker project

The [ue4-docker project](../obtaining-images/ue4-docker) supports running a sandboxed Editor using the [ue4-full container image](https://docs.adamrehn.com/ue4-docker/building-images/available-container-images#ue4-full). Once you have obtained this container image then you can run containers using the commands from the [Implementation-agnostic commands](#implementation-agnostic-commands) section.

### Using container images built from custom Dockerfiles

Check out the [Tips for working with the NVIDIA Container Toolkit](../obtaining-images/write-your-own#tips-for-working-with-the-nvidia-container-toolkit) section of the custom Dockerfile guide for instructions on enabling X11 and/or audio output support in your own container images. Once you have built your container images then you can run them using the commands from the [Implementation-agnostic commands](#implementation-agnostic-commands) section.


## Related media

{% include related-items/media.html url=page.url %}
