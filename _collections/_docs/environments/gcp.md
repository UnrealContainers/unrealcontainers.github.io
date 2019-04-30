---
layout: default
title: Google Cloud Platform (GCP)
tagline: Configure a development or production environment running on GCP.
quickstart: ["build", "run"]
order: 2
---

## Contents
{:.no_toc}

* TOC
{:toc}


## Configure a Compute Engine VM instance as a Windows container host

#### Selecting a machine type

Any Compute Engine machine type with a sufficient amount of memory can be used to run a Windows container host. For building and running Unreal Engine containers a [standard machine type](https://cloud.google.com/compute/docs/machine-types#standard_machine_types) with a minimum of 8GB of memory is recommended.

#### Selecting a virtual machine image

Google provides preconfigured VM images for Windows Server that come with Docker EE preinstalled and are named with the suffix *"for Containers"*:

- Windows Server 2019 Datacenter for Containers
- Windows Server 2019 Datacenter Core for Containers
- Windows Server version 1803 Datacenter Core for Containers
- Windows Server version 1809 Datacenter Core for Containers

After launching an instance using an image that includes Docker EE it is recommended that you update Docker by following the [official update instructions](https://docs.docker.com/install/windows/docker-ee/#update-docker-engine---enterprise).

#### Configuring the instance

Once you have launched an instance and ensured your Docker EE installation is up-to-date, configure the Docker daemon by following the instructions in the [Configuring Docker for building and running Windows containers](./local-windows-server#configuring-docker-for-building-and-running-windows-containers) section of the Windows Server environment configuration page.


## Configure a Compute Engine VM instance as a Linux container host

### Configure a CPU-only instance

#### Selecting a machine type

Any Compute Engine machine type with a sufficient amount of memory can be used to run a Linux container host without [NVIDIA Docker](../concepts/nvidia-docker) support. For building and running Unreal Engine containers a [standard machine type](https://cloud.google.com/compute/docs/machine-types#standard_machine_types) with a minimum of 8GB of memory is recommended.

#### Selecting a virtual machine image

Google does not currently provide preconfigured VM images for Linux with Docker CE preinstalled, so you will need to instead select a base image for one of Docker's [supported Linux distributions](https://docs.docker.com/install/#supported-platforms):

- CentOS 7
- Debian GNU/Linux 9 (stretch)
- Ubuntu 18.04 LTS
- Ubuntu 18.04 LTS Minimal

#### Configuring the instance

Once you have launched an instance running a supported Linux distribution then you will need to install Docker CE by following the instructions in the [Installing and configuring Docker](./local-linux#installing-and-configuring-docker) section of the Linux environment configuration page.

### Configure a GPU-enabled instance with NVIDIA Docker

#### Selecting a machine type

Compute Engine allows you to [add GPU devices](https://cloud.google.com/compute/docs/gpus/add-gpus) to any machine type. For building and running Unreal Engine containers a [standard machine type](https://cloud.google.com/compute/docs/machine-types#standard_machine_types) with a minimum of 8GB of memory is recommended. It is worth noting that GPUs are not available in all Compute Engine [regions and zones](https://cloud.google.com/compute/docs/regions-zones/).

#### Selecting a virtual machine image

As discussed in the [Configure a CPU-only instance](#configure-a-cpu-only-instance) section above, you will need to select a VM image for one of Docker's [supported Linux distributions](https://docs.docker.com/install/#supported-platforms).

#### Installing the NVIDIA drivers

If the VM image you selected for your instance does not include the appropriate NVIDIA GPU drivers then you will need to [install or update the drivers manually](https://cloud.google.com/compute/docs/gpus/add-gpus#install-gpu-driver).

#### Installing Docker and NVIDIA Docker

Once you have ensured your instance is running the appropriate NVIDIA GPU drivers then you will need to install Docker CE and the NVIDIA Docker runtime by following the instructions provided by the [Linux environment configuration page](./local-linux).

#### Optional: configuring PulseAudio support

{% include fragments/pulseaudio.md host="instance" %}
