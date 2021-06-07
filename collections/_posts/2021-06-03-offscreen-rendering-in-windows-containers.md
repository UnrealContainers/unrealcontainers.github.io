---
layout: post
title: Offscreen rendering in Windows containers
author: Adam Rehn and Luke Bermingham
updated: 2021-06-07
tagline: An overview of performing offscreen rendering with Unreal Engine projects in Windows containers.
---

## Overview
{:.no_toc}

GPU accelerated Linux containers have been available for many years thanks to technologies such as the [NVIDIA Container Toolkit](../../docs/concepts/nvidia-docker), and the Unreal Containers community hub features a great deal of information about performing rendering with Unreal Engine projects in Linux containers. Support for [GPU acceleration in Windows containers](../../docs/concepts/gpu-acceleration#gpu-support-for-windows-containers) is far newer and the use of the Unreal Engine for rendering inside Windows containers has received relatively little attention by the authors of the community hub documentation. This blog post aims to address this gap by providing an overview of how to perform offscreen rendering with Unreal projects inside both GPU accelerated Windows containers and inside regular Windows containers through the use of software rendering.

***Special thanks to the team at Epic Games for their feedback on this blog post and for providing the authors with details of the Unreal Engine's support for the Windows Advanced Rasterization Platform (WARP).***

{% include alerts/info.html title="Correction: DirectX Raytracing (DXR) support" content="The original wording of this blog post indicated that the Unreal Engine's DirectX 12 raytracing support did not function correctly inside GPU accelerated Windows containers and caused the Engine to crash due to an assertion failure when it was enabled. The authors have subsequently discovered that this was caused by the absence of the required library files `dxcompiler.dll` and `dxil.dll` from the [DirectX Shader Compiler project](https://github.com/microsoft/DirectXShaderCompiler), and that raytracing functions as expected once these files are present in the container. The example Dockerfile code and surrounding text have been revised to reflect this information." %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Preparing your Unreal project

By default, Unreal Engine projects packaged for Windows are built using the [WINDOWS subsystem rather than the CONSOLE subsystem](https://docs.microsoft.com/en-us/cpp/build/reference/subsystem-specify-subsystem), which means that they behave like GUI applications rather than command-line applications. This is problematic when attempting to run packaged projects inside Windows containers, since they do not display any log output and cannot be controlled properly from a command prompt. You can instruct [UnrealBuildTool](https://docs.unrealengine.com/4.26/en-US/ProductionPipelines/BuildTools/UnrealBuildTool/) to build an additional executable with the CONSOLE subsystem by specifying the `bBuildAdditionalConsoleApp` configuration value in your project's `.Target.cs` file like so:

{% highlight csharp %}
using UnrealBuildTool;
using System.Collections.Generic;

// Note that this is the standalone target file for the project (e.g. `MyProject.Target.cs`),
// NOT the Editor target file for the project (e.g. `MyProjectEditor.Target.cs`)
public class MyProjectTarget : TargetRules
{
	public MyProjectTarget(TargetInfo Target) : base(Target)
	{
		Type = TargetType.Game;
		DefaultBuildSettings = BuildSettingsVersion.V2;
		ExtraModuleNames.AddRange( new string[] { "MyProject" } );
		
		// This instructs UBT to build an additional executable with the CONSOLE subsystem
		bBuildAdditionalConsoleApp = true;
		
		// This enables log output in builds packaged in the Shipping configuration, which is handy
		// if you don't want to package in the Development configuration just to see log output
		bUseLoggingInShipping = true;
	}
}
{% endhighlight %}

When you package your project, an additional executable with the suffix `-Cmd` will be produced alongside the regular binary. For example, if your project is called "MyProject" and you packaged it in the Development configuration then you will see two executables named as follows:

- `MyProject.exe`: this is the executable built with the WINDOWS subsystem, and **is not suitable for use in a container.**

- `MyProject-Cmd.exe` this is the executable built with the CONSOLE subsystem, and **is suitable for use in a container.**


## Creating a runtime container image

Performing rendering with the Unreal Engine inside a Windows container requires both the Visual C++ runtime files and a number of DirectX runtime files that are not included in any of the available Windows base images. As such, these files will need to be copied into the base image of your choice in order to build a suitable [development image or runtime image](../../docs/concepts/image-types).

Although [the Microsoft documentation states](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/gpu-acceleration) that only the [full Windows base image](https://hub.docker.com/_/microsoft-windows) and [the upcoming Windows Server base image](https://techcommunity.microsoft.com/t5/containers/announcing-a-new-windows-server-container-image-preview/ba-p/2304897#toc-hId--1609476460) are officially supported for GPU acceleration, this is not a technical limitation and testing confirms that GPU acceleration also functions correctly in the significantly smaller [Windows Server Core base image](https://hub.docker.com/_/microsoft-windows-servercore). (Code in Microsoft's hcsshim project even suggests [that GPU acceleration should function in Nano Server containers](https://github.com/microsoft/hcsshim/blob/v0.8.17/test/cri-containerd/container_virtual_device_test.go#L166-L174), although the [Windows Nano Server base image](https://hub.docker.com/_/microsoft-windows-nanoserver) lacks so many components required by the Unreal Engine that it would not be practical to use it for rendering.) Since the full Windows base image contains only a handful of DLL files needed by Unreal Engine that are not present in the Windows Server Core base image, **we recommend copying the files into a Server Core image to ensure the final container image size is as small as possible.**

The example Dockerfile below demonstrates how to copy the necessary files into a Windows Server Core base image to create an Unreal Engine runtime image. Note that the variable `BASETAG` represents the tag for the appropriate Windows release (`1909`, `2004`, `20H2`, etc.), which should be specified using a [Docker build argument](https://docs.docker.com/engine/reference/builder/#arg):

{% highlight powershell %}
# escape=`
ARG BASETAG
FROM mcr.microsoft.com/windows:${BASETAG} AS full

# Gather the system DLLs that we need from the full Windows base image
RUN xcopy /y C:\Windows\System32\dsound.dll C:\GatheredDlls\ && `
	xcopy /y C:\Windows\System32\opengl32.dll C:\GatheredDlls\ && `
	xcopy /y C:\Windows\System32\glu32.dll C:\GatheredDlls\ && `
	xcopy /y C:\Windows\System32\MF.dll C:\GatheredDlls\ && `
	xcopy /y C:\Windows\System32\MFPlat.dll C:\GatheredDlls\ && `
	xcopy /y C:\Windows\System32\MFReadWrite.dll C:\GatheredDlls\ && `
	xcopy /y C:\Windows\System32\msdmo.dll C:\GatheredDlls\ && `
	xcopy /y C:\Windows\System32\dxva2.dll C:\GatheredDlls\

# Retrieve the DirectX runtime files required by the Unreal Engine,
# since even the full Windows base image does not include them
RUN curl --progress -L "https://download.microsoft.com/download/8/4/A/84A35BF1-DAFE-4AE8-82AF-AD2AE20B6B14/directx_Jun2010_redist.exe" --output %TEMP%\directx_redist.exe && `
	start /wait %TEMP%\directx_redist.exe /Q /T:%TEMP%\DirectX && `
	expand %TEMP%\DirectX\APR2007_xinput_x64.cab -F:xinput1_3.dll C:\GatheredDlls\ && `
	expand %TEMP%\DirectX\Jun2010_D3DCompiler_43_x64.cab -F:D3DCompiler_43.dll C:\GatheredDlls\ && `
	expand %TEMP%\DirectX\Feb2010_X3DAudio_x64.cab -F:X3DAudio1_7.dll C:\GatheredDlls\ && `
	expand %TEMP%\DirectX\Jun2010_XAudio_x64.cab -F:XAPOFX1_5.dll C:\GatheredDlls\ && `
	expand %TEMP%\DirectX\Jun2010_XAudio_x64.cab -F:XAudio2_7.dll C:\GatheredDlls\

# Retrieve the DirectX shader compiler files needed for DirectX Raytracing (DXR)
RUN curl --progress -L "https://github.com/microsoft/DirectXShaderCompiler/releases/download/v1.6.2104/dxc_2021_04-20.zip" --output %TEMP%\dxc.zip && `
	powershell -Command "Expand-Archive -Path \"$env:TEMP\dxc.zip\" -DestinationPath $env:TEMP" && `
	xcopy /y %TEMP%\bin\x64\dxcompiler.dll C:\GatheredDlls\ && `
	xcopy /y %TEMP%\bin\x64\dxil.dll C:\GatheredDlls\

# Copy the required DLLs from the full Windows base image into a smaller Windows Server Core base image
ARG BASETAG
FROM mcr.microsoft.com/windows/servercore:${BASETAG}
COPY --from=full C:\GatheredDlls\ C:\Windows\System32\

# Install the Visual C++ runtime files using Chocolatey
RUN powershell -NoProfile -ExecutionPolicy Bypass -Command "iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))"
RUN choco install -y vcredist-all
{% endhighlight %}

Once you have built the runtime container image then it can be used for rendering in packaged Unreal Engine projects with either GPU acceleration or software rendering.


## Recommended command-line flags

When running packaged Unreal Engine applications inside Windows containers, we recommend specifying the following command-line flags:

- `-stdout` and `-FullStdOutLogOutput`: these flags ensure all log output is sent to standard output and thus printed in the container's output.

- `-unattended`: this flag suppresses the generation of dialog boxes in the event that an error is encountered. This is extremely important when running the Unreal Engine inside a Windows container because calls to GUI-related Windows API functions such as [DialogBoxW](https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-dialogboxw) and [MessageBoxW](https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-messageboxw) will hang indefinitely inside a containerised environment, blocking execution until the Unreal Engine process or the container itself is forcibly stopped.

- `-RenderOffscreen`: this flag enables offscreen rendering mode, which is required when there is no physical display device.


## Rendering with GPU acceleration

To start a GPU accelerated container for performing rendering with a packaged Unreal project, run the following command:

{% highlight bash %}
# Replace "C:\path\to\project" with the path to a packaged Unreal project
# Replace "my-image" with the tag for the container image you built in the earlier section
docker run --rm -ti --isolation process --device class/5B45201D-F2F2-4F3B-85BB-30FF1F953599 -v "C:\path\to\project:C:\hostdir" "my-image" cmd
{% endhighlight %}

This will start an interactive container with GPU acceleration enabled and the host filesystem directory for your packaged Unreal project bind-mounted to the path `C:\hostdir`. To run your project with offscreen rendering, invoke the `-Cmd` suffixed version of the executable like so:

{% highlight powershell %}
# Start the packaged project with offscreen rendering
C:\hostdir\WindowsNoEditor\MyProject\Binaries\Win64\MyProject-Cmd.exe -stdout -FullStdOutLogOutput -unattended -RenderOffscreen
{% endhighlight %}

You should see log output indicating that the project has loaded and is rendering with DirectX using the GPU from your host system. Note that you may see log output lines starting with the text `LogRHI: Display: New Graphics PSO` followed by a large block of numbers. These messages are normal and can be safely ignored.


## Software rendering with WARP

When running without GPU acceleration, Windows containers can still leverage the [Windows Advanced Rasterization Platform (WARP)](https://docs.microsoft.com/en-us/windows/win32/direct3darticles/directx-warp) to perform software rendering. Although performance will be insufficient for real-time applications, software rendering may still be useful for Unreal projects that render individual frames for linear media or non-interactive use. The Unreal Engine has included support for using WARP with the Direct3D 12 rendering backend [since version 4.14.0](https://github.com/EpicGames/UnrealEngine/blob/4.14.0-release/Engine/Source/Runtime/D3D12RHI/Private/D3D12RHIPrivate.h#L117-L122).

To start a regular Windows container for performing software rendering with a packaged Unreal project, run the following command:

{% highlight bash %}
# Replace "C:\path\to\project" with the path to a packaged Unreal project
# Replace "my-image" with the tag for the container image you built in the earlier section
# Note that there is no `--device` flag here since we are not using GPU acceleration
docker run --rm -ti --isolation process -v "C:\path\to\project:C:\hostdir" "my-image" cmd
{% endhighlight %}

This will start an interactive container with the host filesystem directory for your packaged Unreal project bind-mounted to the path `C:\hostdir`. To run your project with offscreen rendering using WARP, invoke the `-Cmd` suffixed version of the executable like so:

{% highlight powershell %}
# Start the packaged project with offscreen rendering using WARP
# Note that we force DirectX 12 mode, since the Unreal Engine does not support WARP with DirectX 11
C:\hostdir\WindowsNoEditor\MyProject\Binaries\Win64\MyProject-Cmd.exe -stdout -FullStdOutLogOutput -unattended -RenderOffscreen -dx12 -WARP
{% endhighlight %}

You should see log output indicating that the project has loaded and is rendering with DirectX using the device `Microsoft Basic Render Driver`. You may see the same log output lines starting with the text `LogRHI: Display: New Graphics PSO` followed by a large block of numbers that can be seen when running with GPU acceleration, which can be safely ignored.
