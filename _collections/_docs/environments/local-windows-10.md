---
layout: default
title: Local Development (Windows 10)
tagline: Configure a local Windows 10 development environment.
quickstart: ["build"]
order: 6
---

{% capture _alert_content %}
- 64-bit Windows 10 Pro, Enterprise, or Education (Version 1607 or newer)
- Hardware-accelerated virtualisation enabled in the system BIOS/EFI
- [Docker Desktop for Windows](https://docs.docker.com/docker-for-windows/)
{% endcapture %}
{% include alerts/required.html content=_alert_content %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Blacklisted versions of Windows 10

{% include alerts/warning.html content="**Do not use Windows 10 version 1903 or Windows 10 version 1909 to build Windows container images.** There is a known bug in Windows Server versions 1903 and 1909 and Windows 10 versions 1903 and 1909 that [prevents Docker from building images larger than the default limit of 20GB](https://github.com/docker/for-win/issues/4100). These versions of Windows should be avoided when building Unreal Engine container images." %}


## Optimal performance warning for Windows containers

{% include alerts/warning.html content="If you are using Windows 10 to build and run [Windows containers](../concepts/windows-containers) then Docker Desktop for Windows will use Hyper-V isolation mode by default, which suffers from a number of [performance and stability issues](../concepts/windows-containers#hyper-v-isolation-mode-issues) that make it poorly suited for use with Unreal Engine containers. If you are running Windows 10 version 1809 or newer then it is strongly recommended that you [instruct Docker to use process isolation mode instead](https://docs.microsoft.com/en-us/virtualization/windowscontainers/about/faq#can-i-run-windows-containers-in-process-isolated-mode-on-windows-10-enterprise-or-professional). If you are running an older version of Windows 10 that does not support this feature then it is recommended that you update to Windows 10 version 1809 or newer, or alternatively use [Windows Server](./local-windows-server) to build and run Windows containers in process isolation mode." %}


## Installing Docker

Download the [Docker Desktop for Windows installer](https://hub.docker.com/editions/community/docker-ce-desktop-windows) and follow the [installation instructions](https://docs.docker.com/docker-for-windows/install/). Once the application is installed and running it will automatically create Hyper-V virtual machines suitable for running both Windows containers and Linux containers. Unless you have enabled support for the experimental [Linux containers on Windows (LCOW)](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/linux-containers#linux-containers-with-hyper-v-isolation) feature, the two container platforms are treated as separate "modes" that Docker Desktop for Windows can [switch between](https://docs.docker.com/docker-for-windows/#switch-between-windows-and-linux-containers). The default mode after installation is Linux container mode.


## Configuring Docker for building and running Windows containers

{% include alerts/info.html content="Remember to [switch to Windows containers mode](https://docs.docker.com/docker-for-windows/#switch-between-windows-and-linux-containers) before following these instructions." %}

By default, Docker Desktop for Windows imposes a 20GB size limit on container images, which is too low for building and running Unreal Engine containers. You will need to follow [the instructions provided by Microsoft](https://docs.microsoft.com/en-us/visualstudio/install/build-tools-container#step-4-expand-maximum-container-disk-size) to increase the maximum container disk size. The 120GB limit specified in the instructions is sufficient for building and running Unreal Engine container images that do not include the Engine Tools, but a limit of 300GB is recommended for building container images that do include the Engine Tools.


## Configuring Docker for building and running Linux containers

By default, Docker Desktop for Windows will use a Linux virtual machine running the [Moby](https://mobyproject.org/) VM image to run the Docker daemon. This is the configuration that [Microsoft currently recommends](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/linux-containers#when-to-use-moby-vm-vs-lcow) due to the experimental nature of LCOW. The default configured resource limits for the Moby VM are too low for building and running Unreal Engine containers. You will need to use [the Advanced tab of the Docker settings pane](https://docs.docker.com/docker-for-windows/#advanced) to increase the resource allocations:

- Set the CPU count and memory allocation to appropriate levels. A minimum of 8GB of memory is recommended, but more is better.
- Increase the maximum VM disk image size in order to accommodate large container images. If you are building Unreal Engine container images that include the Engine Tools then a minimum disk size of 200GB is recommended.
