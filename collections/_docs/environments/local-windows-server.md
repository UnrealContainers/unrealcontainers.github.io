---
layout: default
title: Local Development (Windows Server)
tagline: Configure a local Windows Server development environment.
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

{% include alerts/warning.html content="**Do not use Windows Server version 1903 or Windows Server version 1909 to build Windows container images with Docker version 19.03.5 or older.** There is a known bug in Windows Server versions 1903 and 1909 and Windows 10 versions 1903 and 1909 that [prevents Docker from building images larger than the default limit of 20GB](https://github.com/docker/for-win/issues/4100). A workaround for this bug was [introduced in Docker version 19.03.6](https://github.com/docker/engine/pull/429)." %}


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

By default, Docker EE under Windows Server imposes a 20GB size limit on container images, which is too low for building and running Unreal Engine containers. You will need to increase the maximum container disk size to the recommended limit of 300GB by following the instructions below.

**Step 1:** Stop the Docker EE service:

{% highlight powershell %}
sc.exe stop docker
{% endhighlight %}

**Step 2:** Edit the Docker daemon configuration file (which is located at `%ProgramData%\Docker\config\daemon.json` by default) and add the top-level `storage-opts` entry to any existing JSON configuration data:

{% highlight json %}
{
  "storage-opts": [
    "size=300GB"
  ]
}
{% endhighlight %}

**Step 3:** Start the Docker EE service again:

{% highlight powershell %}
sc.exe start docker
{% endhighlight %}


## Configuring Docker for building and running Linux containers

{% include alerts/warning.html content="Docker EE under Windows Server only supports running Linux containers using the experimental [Linux containers on Windows (LCOW)](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/linux-containers#linux-containers-with-hyper-v-isolation) feature, which relies on Hyper-V isolation mode. Hyper-V isolation mode currently suffers from a number of [performance and stability issues](../concepts/windows-containers#hyper-v-isolation-mode-issues) that make it poorly suited for use with Unreal Engine containers. It is strongly recommended that you use another platform for building and running Linux containers." %}

There are a number of guides available online that discuss enabling LCOW support under Windows server, but no instructions are provided by the official Microsoft documentation. This page will be updated when official documentation on this configuration scenario becomes available.
