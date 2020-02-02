---
layout: default
title: Local Development (Windows Server)
tagline: Configure a local Windows Server development environment.
quickstart: ["build"]
order: 5
---

{% capture _alert_content %}
- Windows Server 2016 or newer
- [Docker Enterprise Edition (EE) for Windows Server](https://docs.docker.com/install/windows/docker-ee/)
{% endcapture %}
{% include alerts/required.html content=_alert_content %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Blacklisted versions of Windows Server

{% include alerts/warning.html content="**Do not use Windows 10 version 1903 or Windows 10 version 1909 to build Windows container images.** There is a known bug in Windows Server versions 1903 and 1909 and Windows 10 versions 1903 and 1909 that [prevents Docker from building images larger than the default limit of 20GB](https://github.com/docker/for-win/issues/4100). These versions of Windows should be avoided when building Unreal Engine container images." %}


## Obtaining Windows Server

If you don't already own a Windows Server license, you can download an evaluation copy of any version of Windows Server in the [Long-Term Servicing Channel (LTSC)](https://docs.microsoft.com/en-us/windows-server/get-started-19/servicing-channels-19#long-term-servicing-channel-ltsc) from the [Microsoft Evaluation Center](https://www.microsoft.com/en-us/evalcenter/) that will be valid for 180 days:

- [Windows Server 2016 Evaluation](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2016)
- [Windows Server 2019 Evaluation](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2019)

Note that evaluation copies are not available for versions of Windows Server in the [Semi-Annual Channel (SAC)](https://docs.microsoft.com/en-us/windows-server/get-started-19/servicing-channels-19#semi-annual-channel). These versions of Windows Server can only be obtained by Visual Studio subscribers or Microsoft volume licensing customers with [Software Assurance](https://www.microsoft.com/en-us/licensing/licensing-programs/software-assurance-default).


## Installing Docker

As per the [Docker EE Windows Server installation instructions](https://docs.docker.com/install/windows/docker-ee/), run the following commands from an elevated PowerShell prompt:

{% highlight powershell %}
# Add the Docker provider to the PowerShell package manager
Install-Module DockerMsftProvider -Force

# Install Docker EE
Install-Package Docker -ProviderName DockerMsftProvider -Force

# Restart the computer to enable the containers feature
Restart-Computer
{% endhighlight %}

Older versions of Windows Server may sometimes encounter a checksum verification error that prevents the installation process from completing successfully. See [this GitHub issue discussing the error](https://github.com/OneGet/MicrosoftDockerProvider/issues/15) for a number of workarounds that may help to resolve the problem.


## Configuring Docker for building and running Windows containers

By default, Docker EE under Windows Server imposes a 20GB size limit on container images, which is too low for building and running Unreal Engine containers. You will need to follow [the instructions provided by Microsoft](https://docs.microsoft.com/en-us/visualstudio/install/build-tools-container#step-4-expand-maximum-container-disk-size) to increase the maximum container disk size. The 120GB limit specified in the instructions is sufficient for building and running Unreal Engine container images that do not include the Engine Tools, but a limit of 300GB is recommended for building container images that do include the Engine Tools.


## Configuring Docker for building and running Linux containers

{% include alerts/warning.html content="Docker EE under Windows Server only supports running Linux containers using the experimental [Linux containers on Windows (LCOW)](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/linux-containers#linux-containers-with-hyper-v-isolation) feature, which relies on Hyper-V isolation mode. Hyper-V isolation mode currently suffers from a number of [performance and stability issues](../concepts/windows-containers#hyper-v-isolation-mode-issues) that make it poorly suited for use with Unreal Engine containers. It is strongly recommended that you use another platform for building and running Linux containers." %}

There are a number of guides available online that discuss enabling LCOW support under Windows server, but no instructions are provided by the official Microsoft documentation. This page will be updated when official documentation on this configuration scenario becomes available.
