---
layout: default
title: Local Development (Linux)
tagline: Configure a local Linux development environment.
order: 4
---

{% capture _alert_content %}
- 64-bit version of one of Docker's [supported Linux distributions](https://docs.docker.com/install/#supported-platforms) (CentOS 7+, Debian 7.7+, Fedora 26+, Ubuntu 14.04+)
- [Docker Community Edition (CE)](https://docs.docker.com/install/)
- Optional: an NVIDIA GPU if you will be using the NVIDIA Container Toolkit
{% endcapture %}
{% include alerts/required.html content=_alert_content %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Installing and configuring Docker

Install Docker using the instructions specific to your Linux distribution:

- [CentOS installation instructions](https://docs.docker.com/install/linux/docker-ce/centos/)
- [Debian installation instructions](https://docs.docker.com/install/linux/docker-ce/debian/)
- [Fedora installation instructions](https://docs.docker.com/install/linux/docker-ce/fedora/)
- [Ubuntu installation instructions](https://docs.docker.com/install/linux/docker-ce/ubuntu/)

Once Docker is installed, it is recommended that you follow the [post-installation steps for Linux](https://docs.docker.com/install/linux/linux-postinstall/) to enable Docker access for a non-root user, since this helps to avoid filesystem permission issues for a number of [use cases](../use-cases/).

Unlike under other host operating systems, there is no resource limit configuration required under Linux because Docker does not impose any arbitrary memory or disk usage limits by default.


## Optional: installing and configuring the NVIDIA Container Toolkit

If your host system has a compatible NVIDIA GPU and you intend to run containers with GPU acceleration via the [NVIDIA Container Toolkit](../concepts/nvidia-docker), follow the [installation instructions for the NVIDIA Container Toolkit](../concepts/nvidia-docker#what-steps-do-i-need-to-follow-to-make-use-of-the-nvidia-container-toolkit) to install and configure GPU acceleration support.
