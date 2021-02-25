---
layout: default
title: Amazon Web Services (AWS)
tagline: Configure a development or production environment running on AWS.
quickstart: ["build", "run"]
order: 1
---

## Contents
{:.no_toc}

* TOC
{:toc}


## Configure an EC2 instance as a Windows container host

#### Selecting an instance type

Any EC2 instance type with a sufficient amount of memory can be used to run a Windows container host. For building and running Unreal Engine containers a [compute optimised instance](https://aws.amazon.com/ec2/instance-types/#Compute_Optimized) with a minimum of 8GB of memory is recommended.

#### Selecting a virtual machine image

AWS provides preconfigured VM images for Windows Server in the [Amazon Machine Image (AMI)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html) format. The official Quick Start AMI catalogue includes images that come with Docker EE preinstalled and are named with the suffix *"with Containers"*:

- Microsoft Windows Server 2016 Base with Containers
- Microsoft Windows Server 2019 Base with Containers
- Microsoft Windows Server 1809 with Containers

After launching an instance using an AMI that includes Docker EE it is recommended that you update Docker by following the [official update instructions](https://docs.docker.com/install/windows/docker-ee/#update-docker-engine---enterprise).

#### Configuring the instance

Once you have launched an instance and ensured your Docker EE installation is up-to-date, configure the Docker daemon by following the instructions in the [Configuring Docker for building and running Windows containers](./local-windows-server#configuring-docker-for-building-and-running-windows-containers) section of the Windows Server environment configuration page.


## Configure an EC2 instance as a Linux container host

### Configure a CPU-only instance

#### Selecting an instance type

Any EC2 instance type with a sufficient amount of memory can be used to run a Linux container host without [NVIDIA Container Toolkit](../concepts/nvidia-docker) support. For building and running Unreal Engine containers a [compute optimised instance](https://aws.amazon.com/ec2/instance-types/#Compute_Optimized) with a minimum of 8GB of memory is recommended.

#### Selecting a virtual machine image

AWS provides preconfigured VM images for Linux in the [Amazon Machine Image (AMI)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html) format. The official Quick Start AMI catalogue does not currently include images with Docker CE preinstalled, so you will need to instead select a base image for one of Docker's [supported Linux distributions](https://docs.docker.com/install/#supported-platforms). The Quick Start AMI catalogue includes multiple versions of Ubuntu, so we recommend selecting the current LTS version:

- Ubuntu Server 18.04 LTS (HVM), SSD Volume Type

#### Configuring the instance

Once you have launched an instance running a supported Linux distribution then you will need to install Docker CE by following the instructions in the [Installing and configuring Docker](./local-linux#installing-and-configuring-docker) section of the Linux environment configuration page.

### Configure a GPU-enabled instance with the NVIDIA Container Toolkit

#### Selecting an instance type

You will need to select one of the EC2 [Accelerated Computing instance types](https://aws.amazon.com/ec2/instance-types/#Accelerated_Computing) that feature one or more NVIDIA GPUs. The P2, P3, and G3 instance types all include at least one NVIDIA GPU. It is worth noting that GPU-enabled instance types are not available in all AWS [regions and availability zones](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html).

#### Selecting a virtual machine image

As discussed in the [Configure a CPU-only instance](#configure-a-cpu-only-instance) section above, you will need to select an AMI for one of Docker's [supported Linux distributions](https://docs.docker.com/install/#supported-platforms). As with CPU-only instances, we recommend selecting the latest LTS version of Ubuntu from the Quick Start AMI catalogue.

#### Installing the NVIDIA drivers

If the AMI you selected for your instance does not include the appropriate NVIDIA GPU drivers then you will need to [install or update the drivers manually](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/install-nvidia-driver.html).

#### Installing Docker and the NVIDIA Container Toolkit

Once you have ensured your instance is running the appropriate NVIDIA GPU drivers then you will need to install Docker CE and the NVIDIA Container Toolkit by following the instructions provided by the [Linux environment configuration page](./local-linux).

#### Optional: configuring X11 support

Configuring an X11 server with support for GPU acceleration in a VM without a desktop environment installed involves a number of steps:

1. Install both X11 and a desktop environment from the system package repositories for your chosen Linux distribution. For example, under Ubuntu you would use the command `sudo apt-get install xauth xorg ubuntu-desktop`.

2. Use the [nvidia-xconfig](https://manpages.debian.org/stretch/nvidia-xconfig/nvidia-xconfig.1.en.html) tool to generate the `/etc/X11/xorg.conf` configuration file using the command `sudo nvidia-xconfig`.

3. Unfortunately the X11 configuration data generated by the nvidia-xconfig tool requires modification before X11 will correctly detect the available NVIDIA GPU devices. Follow the instructions from [this GitHub repository dedicated to X11 configuration under EC2](https://github.com/agisoft-llc/cloud-scripts) to either manually or automatically modify the `/etc/X11/xorg.conf` configuration file.

4. Start the X11 server manually using the command `sudo /usr/lib/xorg/Xorg :0 -ac`. The `-ac` flag disables authentication, ensuring Unreal Engine containers can access the X11 socket without requiring the use of security tokens.

5. Set the `DISPLAY` environment variable to the display identifier you used when starting the X11 server. If you used the command provided in the previous list item then you would set the environment variable using the command `export DISPLAY=:0`.

Once the X11 server is installed, configured and running, you can enable X11 support inside Unreal Engine containers by bind-mounting the X11 socket from the host system. See the [cloud rendering](../use-cases/cloud-rendering) page for further details on enabling X11 support.
