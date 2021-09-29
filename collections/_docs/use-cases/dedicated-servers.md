---
title: Dedicated Servers
tagline: Run Unreal Engine dedicated servers for multiplayer experiences.
quickstart: "run"
order: 7
---

{% capture _alert_content %}
- Any Linux container image with glibc, TLS certificates and a non-root user account
- An environment [configured for running Linux containers](../environments)
{% endcapture %}
{% include alerts/required.html content=_alert_content %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Overview

An Unreal Engine [dedicated server](https://docs.unrealengine.com/4.27/en-US/InteractiveExperiences/Networking/HowTo/DedicatedServers/) is a headless build of an Unreal project designed solely for handling game logic and network replication for multiplayer experiences. Client builds of the project connect to a dedicated server in order to share a common gameplay session.


## Key considerations

- Because dedicated servers do not perform any rendering, they do not require [GPU acceleration](../concepts/gpu-acceleration), or even an [Unreal Engine runtime image](../concepts/image-types). Instead, they can run in any container image that provides the required system libraries and user configuration. For Linux containers, this means the image must include [glibc](https://www.gnu.org/software/libc/) and TLS certificates, and also provide a non-root user account since Unreal Engine applications refuse to run as the root user. The `cc-debian10:nonroot` variant of Google's lightweight [distroless](https://github.com/GoogleContainerTools/distroless) base image is recommended due to its extremely small size and the fact that it provides a preconfigured non-root user account.

- Dedicated servers primarily use the UDP protocol for communicating with clients. The networking stacks provided by container runtimes such as Docker introduce additional overheads that result in increased latency of UDP packets, so it is strongly recommended that you run Linux containers with [host networking mode](https://docs.docker.com/network/host/) enabled in order to ensure the smoothest experience for users. **Windows containers do not support host networking mode and will always suffer from increased UDP latency.**

- Running dedicated servers in Windows containers is not recommended, due to both the UDP latency issues mentioned above and the need for additional configuration steps as discussed in the [Windows container compatibility](#windows-container-compatibility) section below.


## Implementation guidelines

### Creating a Linux container image

#### Copying pre-packaged files into an image

If you have already packaged a dedicated server for Linux on the host system then you can simply copy these files into a [distroless](https://github.com/GoogleContainerTools/distroless) base image like so:

{% highlight dockerfile %}
# Copy the pre-packaged files into a minimal container image
FROM gcr.io/distroless/cc-debian10:nonroot
COPY --chown=nonroot:nonroot ./LinuxServer /home/nonroot/server

# Expose the port that the dedicated server listens on
EXPOSE 7777/udp

# Set the dedicated server as the container's entrypoint
# (Replace "MyProject" with the name of your project)
ENTRYPOINT ["/home/nonroot/server/MyProject/Binaries/Linux/MyProjectServer", "MyProject"]
{% endhighlight %}

Note that the minimal nature of the distroless image means it does not include a shell like [Bash](https://www.gnu.org/software/bash/) to interpret shell scripts, and so the default startup script for the dedicated server (e.g. `MyProjectServer.sh`) cannot be used as the container entrypoint. Instead, the container entrypoint needs to be configured to directly run the underlying executable for the dedicated server, including the command-line flags that would normally be passed to it by the startup script.

#### Performing a multi-stage build

If you would like to build the dedicated server inside a container then you can use a [multi-stage build](https://docs.docker.com/develop/develop-images/multistage-build/) to perform the build process in an Unreal Engine [development image](../concepts/image-types) and then copy the results to a [distroless](https://github.com/GoogleContainerTools/distroless) base image like so:

{% highlight dockerfile %}
# This example uses the official development image for Unreal Engine 4.27, but any development image will work
FROM ghcr.io/epicgames/unreal-engine:dev-4.27 as builder

# Copy the source code for the project into the container
COPY --chown=ue4:ue4 . /tmp/project

# Build and package the dedicated server for the project
# (Replace "MyProject" with the name of your project)
RUN /home/ue4/UnrealEngine/Engine/Build/BatchFiles/RunUAT.sh BuildCookRun \
	-Server -NoClient -ServerConfig=Development \
	-Project=/tmp/project/MyProject.uproject \
	-UTF8Output -NoDebugInfo -AllMaps -NoP4 -Build -Cook -Stage -Pak -Package -Archive \
	-ArchiveDirectory=/tmp/project/Packaged \
	-Platform=Linux

# Copy the packaged files into a minimal container image
FROM gcr.io/distroless/cc-debian10:nonroot
COPY --from=builder --chown=nonroot:nonroot /tmp/project/Packaged/LinuxServer /home/nonroot/project

# Expose the port that the dedicated server listens on
EXPOSE 7777/udp

# Set the dedicated server as the container's entrypoint
# (Replace "MyProject" with the name of your project)
ENTRYPOINT ["/home/nonroot/server/MyProject/Binaries/Linux/MyProjectServer", "MyProject"]
{% endhighlight %}

### Windows container compatibility

#### Setting the application subsystem

As discussed in the blog post [Offscreen rendering in Windows containers](../../blog/offscreen-rendering-in-windows-containers/), Unreal Engine applications built for Windows are designed to act as graphical applications rather than command-line applications, which prevents them from running correctly inside containers. This also applies to dedicated server targets and must be fixed in the same manner as non-server targets, by instructing [UnrealBuildTool](https://docs.unrealengine.com/4.27/en-US/ProductionPipelines/BuildTools/UnrealBuildTool/) to build an additional executable as a command-line application through the `bBuildAdditionalConsoleApp` configuration value in the `.Target.cs` file:

{% highlight csharp %}
using UnrealBuildTool;
using System.Collections.Generic;

// Note that this is the dedicated server target file for the project (e.g. `MyProjectServer.Target.cs`),
// NOT the Editor target file for the project (e.g. `MyProjectEditor.Target.cs`)
// or the standalone game target file for the project (e.g. `MyProject.Target.cs`)
public class MyProjectServerTarget : TargetRules
{
	public MyProjectServerTarget(TargetInfo Target) : base(Target)
	{
		Type = TargetType.Server;
		DefaultBuildSettings = BuildSettingsVersion.V2;
		ExtraModuleNames.AddRange( new string[] { "MyProject" } );
		
		// This instructs UBT to build an additional executable with the CONSOLE subsystem
		bBuildAdditionalConsoleApp = true;
	}
}
{% endhighlight %}

When you package the dedicated server for your project, an additional executable with the suffix `-Cmd` will be produced alongside the regular binary. For example, if your project is called "MyProject" and you packaged it in the Development configuration then you will see two executables named as follows:

- `MyProjectServer.exe`: this is the executable built with the WINDOWS subsystem, and **is not suitable for use in a container.**

- `MyProjectServer-Cmd.exe` this is the executable built with the CONSOLE subsystem, and **is suitable for use in a container.**

#### Required dependencies

Despite not performing any rendering, dedicated servers still require the following [Microsoft Media Foundation](https://docs.microsoft.com/en-us/windows/win32/medfound/microsoft-media-foundation-sdk) DLL files as of Unreal Engine 4.27:

- `mf.dll`
- `mfplat.dll`
- `mfplay.dll`

These DLL files are not present in the [Windows Server Core](https://hub.docker.com/_/microsoft-windows-servercore) base image and will need to be copied from the [Windows](https://hub.docker.com/_/microsoft-windows) base image (prior to Windows Server 2022) or the [Windows Server](https://hub.docker.com/_/microsoft-windows-server) base image (for Windows Server 2022 and newer.)
