---
title: AI and Machine Learning
tagline: Run Unreal Engine simulations alongside machine learning workloads in the cloud.
quickstart: ["run"]
order: 4
---

{% capture _alert_content %}
- Base container image(s) that [support running packaged Unreal projects with GPU acceleration via the NVIDIA Container Toolkit](../obtaining-images/image-sources)
- A Linux environment [configured for running containers via the NVIDIA Container Toolkit](../environments)
{% endcapture %}
{% include alerts/required.html content=_alert_content %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Overview

Generation of training data for machine learning models is the single most common use of the Unreal Engine [within the context of scientific research](https://ue4research.org/publications). The algorithms underlying these models typically rely on GPU-based computation to achieve maximum performance, and GPU-accelerated containers powered by the [NVIDIA Container Toolkit](../concepts/nvidia-docker) are widely used for running machine learning workloads in the cloud. Unreal Engine containers allow simulations to be packaged and deployed alongside the machine learning models that interact with them, using the same familiar technologies and deployment pipeline. Container orchestration frameworks such as Kubernetes can be used to facilitate network-based or IPC-based communication between containers and to perform training or inference at scale.


## Key considerations

- If your simulations are performing rendering in order to transmit the framebuffer results to a machine learning model then be sure to check out the [cloud rendering](./cloud-rendering) page and familiarise yourself with the relevant details. If your simulations are not performing rendering then be sure to run them in headless mode by specifying the `-nullrhi` flag.

- The choice of base image for your containers will depend on your deployment strategy, and also whether your simulations are performing rendering. See the [Deployment strategies](#deployment-strategies) section for a discussion of these base image requirements.


## Implementation guidelines

### Choosing a communication mechanism

There are a number of mechanisms by which Unreal Engine simulations can interact with software that encapsulates machine learning models. The choice of communication mechanism dictates the manner in which both the simulation and model can be packaged and deployed in containers, so developers should consider this carefully when designing new simulations or preparing existing simulations for containerisation.

#### Network-based communication

Network-based communication is by far the most flexible approach, since it allows the simulation and the model to communicate across different containers or even different underlying host systems. Socket-based network communication is supported natively by the Unreal Engine without the need to integrate additional third-party libraries. If you do decide to integrate additional communication middleware then the use of an RPC framework will allow you to design your system using a standard [microservices architecture](./microservices) and leverage microservice-oriented features of container orchestration frameworks such as Kubernetes.

#### IPC-based communication

IPC-based communication mechanisms such as shared memory can provide better performance than network-based communication when transmitting large quantities of data, albeit at the cost of reduced flexibility. Simulations and models communicating this way must be located on the same underlying host system, but they can still be packaged in separate containers that share an IPC namespace via a grouping mechanism such as a [Kubernetes Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/). The Unreal Engine includes [native support for named shared memory](https://docs.unrealengine.com/en-US/API/Runtime/Core/GenericPlatform/FGenericPlatformMemory/index.html), but care must be taken to match the platform-specific implementation details when accessing shared memory in the model software to ensure full compatibility.

#### In-process communication

In-process communication is by far the least flexible and most brittle approach. Not only does in-process communication force the simulation and the model to run inside the same container, it also introduces significant complexities surrounding the integration of the model software into the Unreal Engine. This may involve the integration of third-party libraries and frameworks or even interpreters for complete programming languages. In most cases any performance benefits associated with this approach do not provide sufficient value to outweigh the cost of the engineering effort required to implement and maintain the integration, and as such **this approach is not recommended**.


### Deployment strategies

#### Separate containers, loosely coupled

{% include styled-lists/checks.html list="- **Supported communication mechanisms:** network-based" %}
{% include styled-lists/crosses.html list="- **Unsupported communication mechanisms:** IPC-based, in-process" %}

In this strategy, the simulation and the machine learning model are deployed in separate containers that are not grouped together in any way. This necessitates network-based communication, since the containers may be scheduled on different underlying host systems. The containers use network discovery to identify one another and typically operate in a client-server model.

If the simulation is not performing rendering then its container can use a base image without support for GPU acceleration and can be run on a CPU-only host system, whilst the model container uses a [CUDA](https://hub.docker.com/r/nvidia/cuda/) or [OpenCL](https://hub.docker.com/r/nvidia/opencl/) equipped base image and runs on a host system with one or more GPUs attached. If the simulation is performing rendering then its container will need to use a base image with [OpenGL](https://hub.docker.com/r/nvidia/opengl/) support and run on a GPU-equipped host system.

This strategy is well-suited to scenarios where there exists a one-to-many relationship between a single simulation and multiple machine learning models, such as when multiple autonomous agents are interacting in a single shared virtual environment. Note that this strategy is **not well-suited to scenarios where multiple agents each require rendered frames from a unique camera**, since the GPUs attached to the simulation container will quickly become a bottleneck as the number of connected agents increases. In such scenarios, it is better to run an Unreal Engine [dedicated server](https://docs.unrealengine.com/en-US/Gameplay/Networking/Server/index.html) to coordinate shared state and have it communicate with multiple sets of paired containers that each tightly couple an agent with an Unreal Engine client that performs rendering on its behalf.

#### Separate containers, tightly coupled

{% include styled-lists/checks.html list="- **Supported communication mechanisms:** network-based, IPC-based" %}
{% include styled-lists/crosses.html list="- **Unsupported communication mechanisms:** in-process" %}

In this strategy, the simulation and the machine learning model are deployed in separate containers that are grouped together using a mechanism such as a [Kubernetes Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/). This ensures the containers will be scheduled on the same underlying host system and allows them to share their network and IPC namespaces, facilitating both network-based and IPC-based communication.

The container base image requirements for this strategy are the same as those for the [loosely coupled strategy described above](#separate-containers-loosely-coupled). Although both the simulation and the model will run together on a GPU-equipped host system, the size of the container image for the simulation can still be kept to a minimum by excluding GPU acceleration support if the simulation does not perform rendering.

This strategy is well-suited to scenarios where there exists a one-to-one relationship between simulations and machine learning models, or when each model is coupled with an Unreal Engine client that communicates with a single Unreal Engine [dedicated server](https://docs.unrealengine.com/en-US/Gameplay/Networking/Server/index.html) that coordinates shared state for a simulation.

#### Single container

{% include styled-lists/checks.html list="- **Supported communication mechanisms:** network-based, IPC-based, in-process" %}

In this strategy, the simulation and the machine learning model are deployed together in a single container. Because the processes are running in the same container on the same underlying host system, all forms of communication are supported. However, this strategy also imposes a number of limitations that do not exist when using separate containers:

- This forces a one-to-one relationship between simulation instances and model instances. One-to-many relationships are not supported.

- This violates the guideline that containers should each encapsulate a single concern, which is widely accepted as an industry best practice.

- Modifications to either the simulation or the machine learning model will necessitate a rebuild of the single shared container image.

- If the simulation is performing rendering then the container base image will need to [support both OpenGL and CUDA](https://hub.docker.com/r/nvidia/cudagl/). (NVIDIA does not currently provide a base image that supports both OpenGL and OpenCL, so you will need to configure a base image manually if your machine learning model requires OpenCL.) If the simulation does not perform rendering then a [CUDA](https://hub.docker.com/r/nvidia/cuda/) or [OpenCL](https://hub.docker.com/r/nvidia/opencl/) equipped base image will be sufficient.
