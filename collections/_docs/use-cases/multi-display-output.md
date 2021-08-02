---
title: Multi-Display Output
tagline: Display rendered output from Unreal Engine projects on multiple displays.
quickstart: "run"
order: 9
---

{% capture _alert_content %}
- An [Unreal Engine runtime image](../concepts/image-types) with support for [GPU acceleration](../concepts/gpu-acceleration) and a copy of the X11 runtime libraries
- A [local Linux development environment](../environments/local-linux) configured for running containers with GPU acceleration and running an X11 server
{% endcapture %}
{% include alerts/required.html content=_alert_content %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Overview

On Linux host systems that satisfy the requirements listed at the top of this page, it is possible to run packaged Unreal projects from inside a container with full GPU acceleration and have rendered output displayed on the host system. This is particularly useful when deploying multiple instances of a project to a coordinated set of machines, such as when using the [nDisplay](https://docs.unrealengine.com/4.26/en-US/WorkingWithMedia/nDisplay/) system to display output on an LED wall or a [CAVE](https://en.wikipedia.org/wiki/Cave_automatic_virtual_environment).

{% include alerts/info.html title="nDisplay information coming soon!" content="This page currently includes only basic information about running packaged Unreal Engine projects in containers and displaying the rendered output on host system displays. Linux support for the Unreal Engine's nDisplay system [is slated for introduction in Unreal Engine 4.27](https://portal.productboard.com/epicgames/1-unreal-engine-public-roadmap/c/322-ndisplay-linux-support-experimental) and the contents of this page will be expanded to include information related to using nDisplay in containers once this new version is released." %}


## Key considerations

- This approach only works on machines that satisfy the hardware and software requirements for running [GPU accelerated Linux containers](../concepts/gpu-acceleration). For orchestration purposes, it is often convenient if these machines are also registered as worker nodes in a Kubernetes cluster, which requires additional configuration.

- Because this approach makes use of a bind-mounted X11 socket, an X11 server must be running on each host system. Linux distributions that ship with an alternative display server protocol enabled by default (e.g. [Wayland](https://wayland.freedesktop.org/)) will require additional configuration to enable an X11 user session.

- Runtime container images must include the X11 runtime libraries in order for the Unreal Engine to recognise the bind-mounted X11 socket. If your container images do not include the relevant libraries then the Engine will default to offscreen rendering and no output will be displayed.


## Implementation guidelines

### Implementation-agnostic commands

#### Running a container with Docker

To run an interactive container locally with Docker, use the following command (replace `unreal-project:latest` with the tag for your container image):

{% highlight bash %}
# Starts a container running a packaged Unreal project
# (Note that we assume the project is defined as the container's entrypoint)
docker run --rm -ti --gpus=all -v/tmp/.X11-unix:/tmp/.X11-unix:rw -e DISPLAY "unreal-project:latest" # <args>
{% endhighlight %}

The individual arguments to the [docker run](https://docs.docker.com/engine/reference/run/) command are explained below:

- `--rm`: Removes the container when it stops.

- `-ti`: Starts an interactive session.

- `--gpus=all`: Enables GPU access via the NVIDIA Container Toolkit.

- `-v/tmp/.X11-unix:/tmp/.X11-unix:rw`: Bind-mounts the X11 socket from the host system.

- `-e DISPLAY`: Propagates the `DISPLAY` environment variable from the host system.

- `"unreal-project:latest"`: **Replace this with the tag for your container image.**

- `# <args>`: **Replace this with any command-line arguments to pass to your packaged Unreal project, such as flags for resolution.**

Your packaged Unreal project should start running and displaying output on the host system's display. If the project does not start automatically and an interactive bash shell starts instead then you will need to ensure you set the project's startup script as the container image's [ENTRYPOINT](https://docs.docker.com/engine/reference/builder/#entrypoint) and rebuild the image.

#### Deploying containers with Kubernetes

When deploying a set of containers with Kubernetes, be sure to include the following in the Pod spec template:

{% highlight yaml %}
# Set the DISPLAY environment variable to match the host system
env:
  -
    name: DISPLAY
    value: ":0"

# Mount the X11 socket in the appropriate path inside the container
volumeMounts:
  -
    mountPath: /tmp/.X11-unix
    name: x11-socket

# Source the X11 socket from the appropriate path on the host system
volumes:
  -
    name: x11-socket
    hostPath:
      path: /tmp/.X11-unix
{% endhighlight %}

Depending on the configuration of the Kubernetes cluster to which the containers will be deployed, it may also be necessary to specify [node selectors](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) or [resource requests](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#extended-resources) for GPUs.

### Using container images from the ue4-runtime project

The base images from the [ue4-runtime project](https://hub.docker.com/r/adamrehn/ue4-runtime) whose tags are suffixed with `-virtualgl` include the X11 runtime libraries required in order to display rendered output on a host system. Once you have built a container image that extends one of these base images with the files for your packaged Unreal project then you can run containers using the commands from the [Implementation-agnostic commands](#implementation-agnostic-commands) section.

### Using container images built from custom Dockerfiles

Check out the [Tips for working with the NVIDIA Container Toolkit](../obtaining-images/write-your-own#tips-for-working-with-the-nvidia-container-toolkit) section of the custom Dockerfile guide for instructions on enabling X11 support in your own container images. Once you have built your container images then you can run them using the commands from the [Implementation-agnostic commands](#implementation-agnostic-commands) section.


## Related media

{% include related-items/media.html url=page.url %}
