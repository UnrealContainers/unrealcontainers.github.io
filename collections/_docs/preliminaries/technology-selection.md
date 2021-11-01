---
title: Technology Selection Guide
tagline: When should I use Linux containers or Windows containers? When should I not use containers at all?
order: 2
---

{% capture _alert_content %}
- Linux containers are the preferred deployment technology if they are compatible with your use case.

- Some use cases are better suited to Windows containers rather than Linux containers.

- Some use cases are not suitable for containers at all, and are best served by virtual machines instead.
{% endcapture %}
{% include alerts/overview.html content=_alert_content %}

Before getting started using Unreal Engine containers, it is important to determine whether containers are appropriate for your intended use case, and if so whether Linux containers or Windows containers are most suitable. Follow the guide below to determine which deployment technology is right for you.


## Contents
{:.no_toc}

* TOC
{:toc}


## Deployment technologies

### Virtual machines (VMs)

Virtual machines are a mature and well-understood deployment technology for providing an isolated virtual environment which mimics the characteristics of a regular bare metal machine. Each individual VM presents itself as a complete machine with a a full set of virtual hardware, upon which operating systems and applications can typically run without requiring any modifications to adapt to the environment. Modern [hypervisors](https://en.wikipedia.org/wiki/Hypervisor) are robust and sophisticated, which makes it incredibly difficult for malicious code to escape the isolation of a VM.

The security and compatibility benefits of virtual machines make them suitable for a wide variety of use cases, but their design also incurs virtualisation overheads which result in reduced deployment density and consequently increased costs when compared to more efficient deployment technologies. The fact that operating systems running in VMs typically follow the same boot process that is used when running on a regular bare metal machine also results in slow startup times for applications that are instanced on demand, since the OS must boot before the application can start.

**<span class="icon-paragraph"><span class="icon fas fa-check icon-advantages"></span> Advantages:</span>**

- Compatible with the widest variety of use cases.
- Secure isolation makes it relatively safe to execute untrusted workloads.

**<span class="icon-paragraph"><span class="icon fas fa-times icon-disadvantages"></span> Disadvantages:</span>**

- Lower deployment density than containers and thus increased operational costs.
- Slow startup times.

### Linux containers

Linux containers are a popular and flexible deployment technology for selectively isolating specific elements of a software environment whilst simultaneously sharing others. The defining characteristic of a container is that it shares the underlying operating system kernel with other containers running on the same host system, but other elements (such as filesystems or network interfaces) can be isolated or shared with either the host system or other selected containers. This flexibility makes it possible to deploy applications in configurations which were not previously possible, such as the [Pod model](https://kubernetes.io/docs/concepts/workloads/pods/) used by the Kubernetes container orchestration system, although some applications may need to be modified to adapt to this deployment environment.

The fact that containers share the underlying kernel and hardware with the host system means they do not incur the same overheads as virtual machines, resulting in greater deployment densities and reduced operational costs. However, the process-based isolation mechanisms used by containers are not as foolproof as a hypervisor, and there is a greater possibility of malicious code escaping from a container than from a VM. This means containers are typically not recommended for running untrusted workloads.

**<span class="icon-paragraph"><span class="icon fas fa-check icon-advantages"></span> Advantages:</span>**

- Facilitates new deployment models which are not possible when using VMs.
- Higher deployment density than VMs and thus reduced operational costs.
- Fast startup times.

**<span class="icon-paragraph"><span class="icon fas fa-times icon-disadvantages"></span> Disadvantages:</span>**

- Compatible with fewer use cases than VMs, since some applications may need to be modified for use in containers.
- Relatively unsafe for running untrusted workloads.

### Windows containers

Windows containers are a far newer technology than their more mature Linux counterparts, and the inherent design of the underlying Windows operating system [introduces complexities and limitations](../concepts/windows-containers) that do not apply to Linux containers. Windows containers are designed to mimic the behaviour of Linux containers and share many of their characteristics, but underlying implementation differences reduce the performance of Windows containers and limit some of the flexibility in how certain aspects of the software environment can be shared between containers. Windows containers are [notably less secure than Linux containers in the presence of malicious code](https://unit42.paloaltonetworks.com/windows-server-containers-vulnerabilities/), and so Microsoft also provides a hybrid implementation known as Hyper-V isolation mode which wraps Windows containers in lightweight virtual machines for improved security. **However, Hyper-V isolated containers are unsuitable for running Unreal Engine workloads due to their [performance characteristics](../concepts/windows-containers#hyper-v-isolation-mode-issues) and are excluded from consideration in this guide.**

Windows containers enjoy the same efficiency benefits as Linux containers when compared to virtual machines, although the overheads of running Windows containers are still greater than those of Linux containers. The relative immaturity of support for GPU acceleration in Windows containers introduces complexities and compatibility issues which are not a problem for Linux containers, such as the need to [take additional steps to enable vendor-specific graphics APIs]({{ "/" | relative_url }}blog/enabling-vendor-specific-graphics-apis-in-windows-containers/) and unlock the full feature set and performance of the Unreal Engine. It is also worth noting that Windows containers can only run on Windows host systems, which incurs Windows Server licensing costs when deploying containers either on premises or in the cloud. The need to use Windows for local development and testing also reduces flexibility when compared to Linux containers, which can be run both natively under Linux and also under Windows or macOS through software such as [Docker Desktop](https://www.docker.com/products/docker-desktop).

**<span class="icon-paragraph"><span class="icon fas fa-check icon-advantages"></span> Advantages:</span>**

- Compatible with most deployment models that can be used with Linux containers.
- Higher deployment density than VMs and thus reduced operational costs.
- Fast startup times.

**<span class="icon-paragraph"><span class="icon fas fa-times icon-disadvantages"></span> Disadvantages:</span>**

- Compatible with fewer use cases than VMs, since some applications may need to be modified for use in containers.
- Compatibility requirements for applications that rely on GPU acceleration are far more complex and cumbersome than Linux containers or VMs.
- Increased deployment cost compared to Linux containers due to Windows Server licensing.
- Kernel compatibility across different Windows releases must be taken into account.
- Lack of support for host networking mode leads to increased latency for UDP, adversely affecting real-time protocols such as WebRTC.
- Reduced filesystem performance compared to Linux containers, which makes building container images noticeably slower.
- Reduced compatibility with container tooling compared to Linux containers.
- Reduced flexibility for local development and testing compared to Linux containers.
- Significantly larger container images than Linux containers.
- Extremely unsafe for running untrusted workloads.


## Selection process

Based on the advantages and disadvantages discussed above, the following order of preference is recommended when selecting a deployment technology:

1. **Linux containers:** this is by far the most performant and cost-effective option. If your Unreal Engine application is compatible with Linux containers then you should select them as your deployment technology, and if your application requires only minimal modification to make it compatible with Linux containers then such modifications will likely still be more cost-effective than alternative deployment technologies.

2. **Windows containers:** although Windows containers are more limited than their Linux counterparts, they are still more performant and cost-effective than VMs. If your Unreal Engine application requires functionality which is exclusive to Windows then it is worthwhile testing whether it is compatible with Windows containers rather than immediately selecting virtual machines as your deployment technology.

3. **Virtual machines:** there are some use cases which simply cannot be run in containers, and in these situations virtual machines are the most appropriate deployment technology for workloads running in the cloud. This is the least cost-effective option and should be selected only once you have verified that your Unreal Engine application cannot be modified to make it compatible with containers.

<br>The most effective way to determine whether your Unreal Engine application is compatible with any given deployment technology is to explicitly test it. Linux containers should be tested first, and if your application is not compatible then you should proceed to test each of the other deployment options in turn. To help expedite this process, the section below contains guidance on known compatibility issues for specific Unreal Engine features and use cases, and can be used to determine whether your use case should be tested with each of the deployment options or if some steps can be skipped.


## Compatibility guidance for specific features and use cases

### Datasmith

As of Unreal Engine 4.27, the [Datasmith](https://docs.unrealengine.com/en-US/Engine/Content/Importing/Datasmith/index.html) plugin is only compatible with Windows, due to its reliance on multiple proprietary SDKs for which Linux binaries are not provided by the vendors. If you are running asset automation pipelines using Datasmith then you should test your pipelines for compatibility with Windows containers, and fall back to using Windows VMs only if your pipeline is not compatible with containers.

### Marketplace plugins

If you are using third-party plugins from the [Unreal Engine Marketplace](https://www.unrealengine.com/marketplace/en-US/store) then you may need to manually add Linux to the list of supported target platforms for each plugin, since many plugins will only list the platforms that their developers have explicitly tested them against. Many plugins are compatible with Linux but will refuse to build when packaging a project for Linux because it is not listed as a supported target platform.

To inspect the list of supported target platforms for a plugin, you will need to open the [plugin descriptor file](https://docs.unrealengine.com/en-US/ProductionPipelines/Plugins/#plugindescriptorfiles) (`.uproject` file) in a text editor or IDE and look for either a top-level [SupportedTargetPlatforms](https://docs.unrealengine.com/4.27/en-US/API/Runtime/Projects/FPluginDescriptor/SupportedTargetPlatforms/) entry, or target platform lists specific to each code module listed under the top-level [Modules](https://docs.unrealengine.com/4.27/en-US/API/Runtime/Projects/FPluginDescriptor/Modules/) entry. Both the plugin itself and all of its relevant code modules will need to be configured to enable Linux as a target platform:

- Any code module that lists Linux in its blocklist (specified by the [BlacklistPlatforms](https://docs.unrealengine.com/4.27/en-US/API/Runtime/Projects/FModuleDescriptor/BlacklistPlatforms/) key) has probably been tested against Linux by the developer and found to be incompatible, but it is still worth removing the entry and testing to see if the plugin builds successfully.

- Any code module that lists at least one target platform in its allowlist (specified by the [WhitelistPlatforms](https://docs.unrealengine.com/4.27/en-US/API/Runtime/Projects/FModuleDescriptor/WhitelistPlatforms/) key) will automatically block all other target platforms, so you will simply need to add an entry for Linux.

### MetaHuman characters

As of Unreal Engine 4.27, the characters generated by the [MetaHuman Creator](https://www.unrealengine.com/en-US/metahuman-creator) application [do not support the Vulkan rendering backend](https://docs.unrealengine.com/4.27/en-US/Resources/Showcases/MetaHumans/#futureimprovementsformetahumans) that is used under Linux. If your Unreal Engine application makes use of MetaHuman characters then you will need to use Windows. You should test your application for compatibility with Windows containers first, and fall back to using Windows VMs only if your application is not compatible with containers.

### Pixel Streaming

Starting in Unreal Engine 4.27, the [Pixel Streaming](https://docs.unrealengine.com/en-US/Platforms/PixelStreaming/index.html) system supports both Windows and Linux, and the [official container images](../obtaining-images/official-images) and demos that ship with the Unreal Engine include resources for running Pixel Streaming applications in Linux containers. You should test Pixel Streaming applications for compatibility with Linux containers first, unless your application requires features that only work under Windows (such as ray tracing or MetaHuman characters.)

It is important to note that running Pixel Streaming applications inside Windows containers incurs increased UDP latency (and thus WebRTC stream latency) and [requires experimental functionality to enable hardware video encoding that is not recommended for production use](../use-cases/pixel-streaming#key-considerations). If your Pixel Streaming application is not compatible with Linux containers then it is recommended that you use Windows VMs.

### Ray tracing

As of Unreal Engine 4.27, [real-time ray tracing](https://docs.unrealengine.com/en-US/RenderingAndGraphics/RayTracing/) is not supported under Linux. If your Unreal Engine application requires ray tracing then you will need to use Windows. Fortunately, DirectX Raytracing (DXR) [functions correctly inside Windows containers]({{ "/" | relative_url }}blog/offscreen-rendering-in-windows-containers/), so you should test your application for compatibility with Windows containers first, and fall back to using Windows VMs only if your application is not compatible with containers. (Unless of course you are using Pixel Streaming together with ray tracing, in which case you should use Windows VMs as noted in the section above.)

### Software rendering

As evidenced by the lack of documented options in the [Software rendering in Linux containers](../concepts/gpu-acceleration#software-rendering-in-linux-containers) section of the GPU acceleration page, the authors of this documentation have yet to find a suitable software renderer for Linux that satisfies the requirements of the Unreal Engine's Vulkan rendering backend. If your application requires software rendering (e.g. for non-realtime rendering tasks that can be accomplished without a GPU) then you will need to use Windows, which supports the full Direct3D 11 and Direct3D 12 graphics APIs through the [Windows Advanced Rasterization Platform (WARP)](https://docs.microsoft.com/en-us/windows/win32/direct3darticles/directx-warp).

The Unreal Engine's Direct3D 12 rendering backend includes built-in support for WARP and this configuration [is known to function correctly inside Windows containers]({{ "/" | relative_url }}blog/offscreen-rendering-in-windows-containers/#software-rendering-with-warp), so you should test your application for compatibility with Windows containers and fall back to using Windows VMs only if your application is not compatible with containers.
