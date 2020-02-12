---
layout: default
title: Microsoft Azure
tagline: Configure a development or production environment running on Azure.
quickstart: ["build", "run"]
order: 3
---

## Contents
{:.no_toc}

* TOC
{:toc}


## Configure a virtual machine as a Windows container host

#### Selecting a virtual machine size

Any virtual machine size with a sufficient amount of memory can be used to run a Windows container host. For building and running Unreal Engine containers a size from a [compute optimised series](https://azure.microsoft.com/en-au/pricing/details/virtual-machines/windows/#fsv2-series) with a minimum of 8GB of memory is recommended.

#### Selecting a virtual machine image

Microsoft provides [preconfigured VM images for Windows Server](https://azuremarketplace.microsoft.com/en-us/marketplace/apps/Microsoft.WindowsServer) that come with Docker EE preinstalled and are named with the suffix *"with Containers"*:

- Windows Server 2016 Datacenter - with Containers
- Windows Server 2019 Datacenter with Containers
- Windows Server 2019 Datacenter Server Core with Containers
- Windows Server, version 1803 with Containers
- Windows Server, version 1809 with Containers

After launching a virtual machine using an image that includes Docker EE it is recommended that you update Docker by following the [official update instructions](https://docs.docker.com/install/windows/docker-ee/#update-docker-engine---enterprise).

#### Configuring the virtual machine

Once you have launched a virtual machine and ensured your Docker EE installation is up-to-date, configure the Docker daemon by following the instructions in the [Configuring Docker for building and running Windows containers](./local-windows-server#configuring-docker-for-building-and-running-windows-containers) section of the Windows Server environment configuration page.


## Configure a virtual machine as a Linux container host

### Configure a CPU-only virtual machine

#### Selecting a virtual machine size

Any virtual machine size with a sufficient amount of memory can be used to run a Linux container host without [NVIDIA Container Toolkit](../concepts/nvidia-docker) support. For building and running Unreal Engine containers a size from a [compute optimised series](https://azure.microsoft.com/en-au/pricing/details/virtual-machines/linux/#fsv2-series) with a minimum of 8GB of memory is recommended.

#### Selecting a virtual machine image

Microsoft does not currently provide preconfigured VM images for Linux with Docker CE preinstalled, so you will need to instead select a base image for one of Docker's [supported Linux distributions](https://docs.docker.com/install/#supported-platforms):

- [Ubuntu Server 18.04 LTS](https://azuremarketplace.microsoft.com/en-us/marketplace/apps/Canonical.UbuntuServer1804LTS)

#### Configuring the virtual machine

Once you have launched a virtual machine running a supported Linux distribution then you will need to install Docker CE by following the instructions in the [Installing and configuring Docker](./local-linux#installing-and-configuring-docker) section of the Linux environment configuration page.

### Configure a GPU-enabled virtual machine with the NVIDIA Container Toolkit

#### Selecting a virtual machine size

You will need to select a VM size from a series that features one or more NVIDIA GPUs. The [NC-series](https://azure.microsoft.com/en-au/pricing/details/virtual-machines/linux/#n-series), [NV-series](https://azure.microsoft.com/en-au/pricing/details/virtual-machines/linux/#nv-series), and [ND-series](https://azure.microsoft.com/en-au/pricing/details/virtual-machines/linux/#nd-series) all include at least one NVIDIA GPU. It is worth noting that GPU-enabled virtual machines are not available in all [Azure regions](https://azure.microsoft.com/en-au/global-infrastructure/regions/).

#### Selecting a virtual machine image

As discussed in the [Configure a CPU-only virtual machine](#configure-a-cpu-only-virtual-machine) section above, you will need to select a VM image for one of Docker's [supported Linux distributions](https://docs.docker.com/install/#supported-platforms). As with CPU-only virtual machines, we recommend selecting the latest LTS version of Ubuntu.

#### Installing the NVIDIA drivers

If the image you selected for your virtual machine does not include the appropriate NVIDIA GPU drivers then you will need to [install or update the drivers manually](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/n-series-driver-setup).

#### Installing Docker and the NVIDIA Container Toolkit

Once you have ensured your virtual machine is running the appropriate NVIDIA GPU drivers then you will need to install Docker CE and the NVIDIA Container Toolkit by following the instructions provided by the [Linux environment configuration page](./local-linux).

#### Optional: configuring PulseAudio support

{% include fragments/pulseaudio.md host="virtual machine" %}
