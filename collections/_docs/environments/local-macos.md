---
layout: default
title: Local Development (macOS)
tagline: Configure a local macOS development environment.
quickstart: ["build"]
order: 7
---

{% capture _alert_content %}
- 2010 or newer model Mac hardware
- macOS 10.10.3 Yosemite or newer
- [Docker Desktop for Mac](https://docs.docker.com/docker-for-mac/)
{% endcapture %}
{% include alerts/required.html content=_alert_content %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Installing Docker

Download the [Docker Desktop for Mac installer](https://hub.docker.com/editions/community/docker-ce-desktop-mac) and follow the [installation instructions](https://docs.docker.com/docker-for-mac/install/). Once the application is installed and running it will automatically create a Linux virtual machine running the [Moby](https://mobyproject.org/) VM image, within which the Docker daemon will run.


## Configuring Docker for building and running Linux containers

By default Docker Desktop for Mac will configure the Moby VM with resource limits that are too low for building and running Unreal Engine containers. You will need to use the Docker settings pane to increase the resource allocations:

- Use [the Advanced tab of the Docker settings pane](https://docs.docker.com/docker-for-mac/#advanced) to increase the CPU count and memory allocation to appropriate levels. A minimum of 8GB of memory is recommended.
- Use [the Disk tab of the Docker settings pane](https://docs.docker.com/docker-for-mac/#disk) to increase the maximum VM disk image size in order to accommodate large container images. If you are building Unreal Engine container images that include the Engine Tools then a minimum disk size of 200GB is recommended.
