---
layout: post
title: Enabling vendor-specific graphics APIs in Windows containers
author: Adam Rehn, Luke Bermingham and Aidan Possemiers
updated: 2021-07-23
tagline: A guide on how to enable support for vendor-specific graphics APIs inside GPU accelerated Windows containers.
---

{% include alerts/warning.html title="Experimental use only!" content="Using graphics APIs other than DirectX inside Windows containers is not officially supported by Microsoft or by any of the graphics hardware vendors, and comes with no guarantees regarding functionality or performance. We do not recommend using this feature for production workloads until official support is introduced at some point in the future." %}

## Overview
{:.no_toc}

As per [the official Microsoft documentation](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/gpu-acceleration), GPU accelerated Windows containers only officially support the DirectX graphics API and any frameworks built atop it, such as DirectML. By default, Windows containers cannot make use of other vendor-neutral graphics APIs (such as OpenCL, OpenGL, and Vulkan) or vendor-specific graphics APIs such as AMD AMF or NVIDIA NVENC. Although the rendering and compute functionality of other vendor-neutral graphics APIs is largely covered by DirectX itself, the absence of vendor-specific APIs for tasks such as hardware accelerated video encoding impacts many popular Unreal Engine use cases such as [Pixel Streaming](../../docs/use-cases/pixel-streaming) and has precluded their use in Windows containers until now.

Fortunately, our investigation has discovered that GPU accelerated Windows containers have access to all of the necessary files for accessing other graphics APIs, and enabling support is simply a matter of locating these files at container startup and copying them to the appropriate location. In this blog post we present an extensible approach to enabling additional graphics APIs in Windows containers and provide example code to enable vendor-specific graphics APIs for both AMD and NVIDIA GPUs.

***Special thanks to the team at Epic Games for their feedback on this blog post, to Microsoft for providing insights into the mechanisms by which GPU acceleration functions in Windows containers, and to AMD for their rapid response in addressing the graphics driver bug that we encountered during our investigation.***


## Contents
{:.no_toc}

* TOC
{:toc}


## Anatomy of GPU accelerated Windows containers

### WDDM Architecture

In order to understand how GPUs are accessed from inside Windows containers, it is first necessary to understand how graphics devices are accessed under Windows in general. Since Windows 8, all graphics drivers must conform to the [Windows Display Driver Model (WDDM)](https://docs.microsoft.com/en-us/windows-hardware/drivers/display/windows-vista-display-driver-model-design-guide), which defines how graphics APIs and Windows system services interact with driver code. An overview of the architecture of WDDM is depicted in Figure 1:

{% capture _figure_caption %}
Figure 1: Architecture of the Windows Display Driver Model (WDDM) as it pertains to Windows containers. Adapted from the WDDM architecture diagram in the X.Org Developers Conference 2020 presentation "WSL Graphics Architecture" ([slides](https://xdc2020.x.org/event/9/contributions/610/attachments/700/1295/XDC_-_WSL_Graphics_Architecture.pdf), [video](https://www.youtube.com/watch?v=b2mnbyRgXkY&t=4680s)) and from the architecture diagram on the [Windows Display Driver Model (WDDM) Architecture](https://docs.microsoft.com/en-us/windows-hardware/drivers/display/windows-vista-and-later-display-driver-model-architecture) page of the Windows Driver Documentation, which is Copyright &copy; Microsoft Corporation and is licensed under a [Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/).
{% endcapture %}
{% include figures/post.html image="wddm-architecture.svg" caption=_figure_caption markdown=true %}

The key components depicted in Figure 1 are as follows:

- **Direct3D API**: this is the public Direct3D graphics API, which forms part of DirectX family of APIs and is designed for use by user-facing applications such as the Unreal Engine. The DirectX header files that define this API are available in the [DirectX-Headers](https://github.com/microsoft/DirectX-Headers) repository on GitHub.

- **Other graphics APIs**: all other graphics APIs which are not based on DirectX, such as OpenGL, OpenCL, Vulkan, etc. For a list of common graphics APIs and their relation to Unreal Engine use cases, see the [graphics APIs section of the GPU acceleration page](../../docs/concepts/gpu-acceleration#graphics-apis).

- **WDDM Interface**: this is the set of low-level WDDM APIs that expose kernel graphics functionality to userspace code. The API functions are prefixed with `D3DKMT` and are defined in the header [d3dkmthk.h](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/d3dkmthk/), which is available as part of the [Windows Driver Kit (WDK)](https://docs.microsoft.com/en-us/windows-hardware/drivers/download-the-wdk) and also [as part of the Windows 10 SDK for Windows 10 version 2004 and newer](https://docs.microsoft.com/en-us/windows-hardware/drivers/display/container-non-dx#d3dkmt-headers).

- **DirectX Graphics Kernel**: this is the component of the Windows kernel that implements the underlying functionality exposed by the API functions of the WDDM Interface. The graphics kernel is responsible for ensuring GPU resources can be shared between multiple userspace processes, and performs tasks such as GPU scheduling and memory management. The code for this component is located in the file `dxgkrnl.sys` and in accompanying files with the prefix `dxgmms*.sys` (e.g. `dxgmms1.sys`, `dxgmms2.sys`, etc.)

- **Kernel Mode Driver (KMD)**: this is the component of the graphics driver that runs in kernel mode and communicates directly with hardware devices on behalf of the DirectX Graphics Kernel.

- **User Mode Driver (UMD)**: this is the component of the graphics driver that runs in userspace and provides driver functionality for use by the Direct3D API. The Direct3D API interacts with the low-level WDDM Interface on behalf of the User Mode Driver, so the driver need not interact with it directly.

- **Installable Client Driver (ICD)**: this refers to components of the graphics driver that run in userspace and provide driver functionality for use by all other graphics APIs which are not based on DirectX. There may be a distinct Installable Client Driver for each supported graphics API. Unlike the Direct3D User Mode Driver, an Installable Client Driver directly interacts with the low-level WDDM Interface.

The components depicted in green boxes in the diagram (Kernel Mode Driver, User Mode Driver, Installable Client Driver) are provided by the graphics driver from the hardware vendor. The mechanisms by which these components are located and loaded at runtime are of particular interest when considering GPU access inside Windows containers.

### Loading User Mode Drivers (UMDs) and Installable Client Drivers (ICDs)

Although the Kernel Mode Driver component of a graphics driver is loaded automatically by Windows when initialising a GPU, the accompanying User Mode Driver and any Installable Client Drivers need to be located by each new process that requests access to the graphics device. To do so, applications call the [D3DKMTQueryAdapterInfo](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/d3dkmthk/nf-d3dkmthk-d3dkmtqueryadapterinfo) API function from the low-level WDDM Interface to query the filesystem locations of userspace driver components. The WDDM Interface will then consult the Windows Registry to locate the relevant paths inside the system's [Driver Store](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/driver-store), which acts as the central repository for driver installation files. The Direct3D API performs this call in order to automatically locate the User Mode Driver, as do the runtimes for other vendor-neutral APIs such as OpenGL and OpenCL in order to locate the appropriate Installable Client Driver.

It is important to note that the WDDM Interface contains specific logic to handle cases where the calling process is running inside a Windows container or inside a virtual machine that is accessing the host system's GPU through WDDM GPU Paravirtualization (GPU-PV), and will adjust the returned information accordingly. This is why Microsoft recommends that applications [always call the D3DKMTQueryAdapterInfo function instead of querying the registry or the filesystem directly](https://docs.microsoft.com/en-us/windows-hardware/drivers/display/container-non-dx#driver-modifications-to-registry-and-file-paths), since bypassing the adjustment logic provided by the WDDM Interface will lead to incorrect results inside containers or virtual machines and prevent User Mode Drivers and Installable Client Drivers from being loaded correctly.

### Accessing GPUs inside containers

Processes running inside process-isolated Windows containers interact with the host system kernel in exactly the same manner as processes running directly on the host. As such, processes interact with the WDDM Interface as usual to communicate with the DirectX Graphics Kernel and the Kernel Mode Driver. The key difference is in how User Mode Drivers and Installable Client Drivers are loaded.

Windows container images include their own set of driver packages in the system Driver Store, which is distinct from the host system's Driver Store. When a GPU accelerated Windows container is created by the [Windows Host Compute Service (HCS)](https://techcommunity.microsoft.com/t5/containers/introducing-the-host-compute-service-hcs/ba-p/382332), the host system's Driver Store is automatically mounted in the container's filesystem under the path `C:\Windows\System32\HostDriverStore`. Filesystem paths returned by calls to the [D3DKMTQueryAdapterInfo](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/d3dkmthk/nf-d3dkmthk-d3dkmtqueryadapterinfo) function will be automatically adjusted by the WDDM Interface to reference the mount path for the host system's Driver Store instead of the container's own Driver Store, which ensures User Mode Drivers and Installable Client Drivers will be loaded from the correct location.

Although the automatic adjustment functionality provided by the WDDM Interface ensures the Direct3D API functions correctly inside GPU accelerated Windows containers, an additional step is required in order to support other graphics APIs. The runtime libraries for these APIs must be available for user applications to load before the runtime library can then locate and load the appropriate Installable Client Driver. Most applications expect the runtime libraries to be located in the Windows system directory under `C:\Windows\System32` and in some cases will even refuse to load them from any other location for security reasons.

Windows provides a mechanism for drivers to specify additional runtime libraries that should be automatically copied to the Windows system directory, in the form of the [CopyToVmOverwrite and CopyToVmWhenNewer registry keys](https://docs.microsoft.com/en-us/windows-hardware/drivers/display/container-non-dx#driver-inf-modifications) and their SysWOW64 counterparts. Recent graphics drivers from AMD, Intel and NVIDIA all ship with registry entries to copy the runtime libraries for their supported vendor-neutral graphics APIs, along with a subset of their supported vendor-specific graphics APIs. Unfortunately, as of the time of writing, the automatic file copy feature is only supported for GPU-PV and so process-isolated Windows containers must manually copy the files for runtime libraries to the Windows system directory at startup.


## Approach to enabling additional graphics APIs

As discussed in the previous section, GPU accelerated Windows containers must copy the runtime libraries for non-DirectX graphics APIs to the Windows system directory before these graphics APIs can be used by applications. Ideally this copy operation should be performed when the container first starts, before its entrypoint application begins execution. As per the [Best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#entrypoint) page of the Docker documentation, the most appropriate mechanism for performing this type of container startup task is to create a helper script that acts as the container's entrypoint and wraps the real entrypoint application.

Consider the following Dockerfile, expanded from the blog post [Offscreen rendering in Windows containers](../offscreen-rendering-in-windows-containers/) and modified to include an entrypoint:

{% highlight powershell %}
# escape=`
ARG BASETAG
FROM mcr.microsoft.com/windows:${BASETAG} AS full

# Gather the system DLLs that we need from the full Windows base image
RUN xcopy /y C:\Windows\System32\avicap32.dll C:\GatheredDlls\ && `
	xcopy /y C:\Windows\System32\avrt.dll C:\GatheredDlls\ && `
	xcopy /y C:\Windows\System32\d3d10warp.dll C:\GatheredDlls\ && `
	xcopy /y C:\Windows\System32\D3DSCache.dll C:\GatheredDlls\ && `
	xcopy /y C:\Windows\System32\dsound.dll C:\GatheredDlls\ && `
	xcopy /y C:\Windows\System32\dxva2.dll C:\GatheredDlls\ && `
	xcopy /y C:\Windows\System32\glu32.dll C:\GatheredDlls\ && `
	xcopy /y C:\Windows\System32\mf.dll C:\GatheredDlls\ && `
	xcopy /y C:\Windows\System32\mfplat.dll C:\GatheredDlls\ && `
	xcopy /y C:\Windows\System32\mfplay.dll C:\GatheredDlls\ && `
	xcopy /y C:\Windows\System32\mfreadwrite.dll C:\GatheredDlls\ && `
	xcopy /y C:\Windows\System32\msdmo.dll C:\GatheredDlls\ && `
	xcopy /y C:\Windows\System32\msvfw32.dll C:\GatheredDlls\ && `
	xcopy /y C:\Windows\System32\opengl32.dll C:\GatheredDlls\ && `
	xcopy /y C:\Windows\System32\ResourcePolicyClient.dll C:\GatheredDlls\

# Retrieve the DirectX runtime files required by the Unreal Engine, since even the full Windows base image does not include them
RUN curl --progress -L "https://download.microsoft.com/download/8/4/A/84A35BF1-DAFE-4AE8-82AF-AD2AE20B6B14/directx_Jun2010_redist.exe" --output %TEMP%\directx_redist.exe && `
	start /wait %TEMP%\directx_redist.exe /Q /T:%TEMP%\DirectX && `
	expand %TEMP%\DirectX\APR2007_xinput_x64.cab -F:xinput1_3.dll C:\GatheredDlls\ && `
	expand %TEMP%\DirectX\Feb2010_X3DAudio_x64.cab -F:X3DAudio1_7.dll C:\GatheredDlls\ && `
	expand %TEMP%\DirectX\Jun2010_D3DCompiler_43_x64.cab -F:D3DCompiler_43.dll C:\GatheredDlls\ && `
	expand %TEMP%\DirectX\Jun2010_XAudio_x64.cab -F:XAudio2_7.dll C:\GatheredDlls\ && `
	expand %TEMP%\DirectX\Jun2010_XAudio_x64.cab -F:XAPOFX1_5.dll C:\GatheredDlls\

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
RUN powershell -NoProfile -ExecutionPolicy Bypass -Command "[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))"
RUN choco install -y vcredist-all

# Copy our Unreal Engine application into the container image here
# (This assumes we have a packaged project called "MyProject" with a `-Cmd.exe` suffixed
#  version as per the blog post "Offscreen rendering in Windows containers")
COPY WindowsNoEditor C:\WindowsNoEditor

# Set our Unreal Engine application as the container's entrypoint
ENTRYPOINT ["cmd.exe", "/S", "/C", "C:\\WindowsNoEditor\\MyProject\\Binaries\\Win64\\MyProject-Cmd.exe", "-stdout", "-FullStdOutLogOutput", "-RenderOffscreen", "-unattended", "-ResX=1024", "-ResY=768"]
{% endhighlight %}

We want our helper script to wrap the real entrypoint in the most transparent fashion possible, so that we can invoke it by adding a single element to the entrypoint command array:

{% highlight dockerfile %}
# Copy the helper script (and accompanying PowerShell script) into the container image
COPY entrypoint.cmd C:\entrypoint.cmd
COPY enable-graphics-apis.ps1 C:\enable-graphics-apis.ps1

# Set our Unreal Engine application as the container's entrypoint, wrapped in the helper script
ENTRYPOINT ["cmd.exe", "/S", "/C", "C:\\entrypoint.cmd", "C:\\WindowsNoEditor\\MyProject\\Binaries\\Win64\\MyProject-Cmd.exe", "-stdout", "-FullStdOutLogOutput", "-RenderOffscreen", "-unattended", "-ResX=1024", "-ResY=768"]
{% endhighlight %}

To achieve this, we can populate `entrypoint.cmd` as follows:

{% highlight batch %}
@echo off

@rem Enable vendor-specific graphics APIs if the container is running with GPU acceleration
powershell -ExecutionPolicy Bypass -File "%~dp0.\enable-graphics-apis.ps1"

@rem Run the entrypoint command specified via our command-line parameters
%*
{% endhighlight %}

This will run a PowerShell script to copy the runtime libraries for the vendor-specific graphics APIs that we wish to enable to the Windows system directory, and then run whatever command was specified after the helper script in the container's entrypoint (in this case our Unreal Engine application.) The code for performing the copy is placed in `enable-graphics-apis.ps1`, which needs to perform the following steps:

- Identify which graphics drivers are present in the Driver Store bind-mounted from the host system. Since GPU accelerated Windows containers support any graphics driver that is compliant with WDDM version 2.5 or newer, the container won't know until runtime what graphics device is being used and which hardware vendor manufactured the device.

- Locate the runtime libraries for each of the vendor-specific graphics APIs that we wish to enable.

- Copy the runtime libraries to the Windows system directory, optionally renaming the files since their filenames in the Driver Store may be different to the filenames that applications expect to find in the system directory. (This renaming feature is also present in the automatic file copy feature discussed in the section above, and the registry keys that ship with graphics drivers often specify a mapping for renaming their runtime library DLL files.)

To support the ability to rename files as they are copied, we can use a function like this:

{% highlight powershell %}
# Copies the supplied list of files from the specified source directory to the System32 directory
function CopyToSystem32($sourceDirectory, $filenames, $rename)
{
	foreach ($filename in $filenames)
	{
		# Determine whether we are renaming the file when we copy it to the destination
		$source = "$sourceDirectory\$filename"
		$destination = "C:\Windows\System32\$filename"
		if ($rename -and $rename[$filename])
		{
			$renamed = $rename[$filename]
			$destination = "C:\Windows\System32\$renamed"
		}
		
		# Perform the copy
		Write-Host "    Copying $source to $destination"
		try {
			Copy-Item -Path "$source" -Destination "$destination" -ErrorAction Stop
		}
		catch {
			Write-Host "    Warning: failed to copy file $filename" -ForegroundColor Yellow
		}
	}
}
{% endhighlight %}

This function can then be called by specific blocks of code for each graphics hardware vendor. Example blocks of code for AMD and NVIDIA are provided in the sections that follow.


## Enabling AMD graphics APIs

{% capture _alert_content %}
In order to enable AMD graphics APIs in Windows containers, the host system must be running [Radeon Adrenalin 21.7.1](https://www.amd.com/en/support/kb/release-notes/rn-rad-win-21-7-1) or newer, which was released on the 15th of July 2021. Older versions of the AMD graphics drivers for Windows contain a bug which prevents them from working correctly inside GPU accelerated Windows containers.
{% endcapture %}
{% include alerts/warning.html title="Minimum driver version required!" content=_alert_content %}

To enable support for AMD graphics APIs, add the following code to the PowerShell script:

{% highlight powershell %}
# Attempt to locate the AMD Display Driver directory in the host system's driver store
$amdSentinelFile = (Get-ChildItem "C:\Windows\System32\HostDriverStore\FileRepository\u*.inf_amd64_*\*\aticfx64.dll" -ErrorAction SilentlyContinue)
if ($amdSentinelFile) {
	
	# Retrieve the path to the directory containing the DLL files for AMD graphics APIs
	$amdDirectory = $amdSentinelFile[0].VersionInfo.FileName | Split-Path
	Write-Host "Found AMD Display Driver directory: $amdDirectory"
	
	# Copy the DLL files for the AMD DirectX drivers to System32
	# (Note that copying these files to System32 is not necessary for rendering,
	#  but applications using ADL may attempt to load these files from System32)
	Write-Host "`nCopying AMD DirectX driver files:"
	CopyToSystem32 `
		-SourceDirectory $amdDirectory `
		-Filenames @(`
			"aticfx64.dll", `
			"atidxx64.dll"
		)

	# Copy the DLL files needed for AMD Display Library (ADL) support to System32
	Write-Host "`nEnabling AMD Display Library (ADL) support:"
	CopyToSystem32 `
		-SourceDirectory $amdDirectory `
		-Filenames @(`
			"atiadlxx.dll", `
			"atiadlxy.dll" `
		)
	
	# Copy the DLL files needed for AMD Advanced Media Framework (AMF) support to System32
	Write-Host "`nEnabling AMD Advanced Media Framework (AMF) support:"
	CopyToSystem32 `
		-SourceDirectory $amdDirectory `
		-Filenames @(`
			"amfrt64.dll", `
			"amfrtdrv64.dll", `
			"amdihk64.dll" `
		)
	
	# Print a blank line before any subsequent output
	Write-Host ""
}
{% endhighlight %}

This will locate and copy the necessary files in order to enable the following APIs:

- [AMD Display Library (ADL)](https://gpuopen.com/adl/): provides low-level display driver functionality. This API is used by the Unreal Engine for optimising performance on AMD GPUs under Windows.

- [Advanced Media Framework (AMF)](https://gpuopen.com/advanced-media-framework/): provides hardware accelerated video encoding and decoding functionality. This API is used by the Unreal Engine for Pixel Streaming.


## Enabling NVIDIA graphics APIs

To enable support for NVIDIA graphics APIs, add the following code to the PowerShell script:

{% highlight powershell %}
# Attempt to locate the NVIDIA Display Driver directory in the host system's driver store
$nvidiaSentinelFile = (Get-ChildItem "C:\Windows\System32\HostDriverStore\FileRepository\nv*.inf_amd64_*\nvapi64.dll" -ErrorAction SilentlyContinue)
if ($nvidiaSentinelFile) {
	
	# Retrieve the path to the directory containing the DLL files for NVIDIA graphics APIs
	$nvidiaDirectory = $nvidiaSentinelFile[0].VersionInfo.FileName | Split-Path
	Write-Host "Found NVIDIA Display Driver directory: $nvidiaDirectory"
	
	# Copy the DLL file for NVAPI to System32
	Write-Host "`nEnabling NVIDIA NVAPI support:"
	CopyToSystem32 `
		-SourceDirectory $nvidiaDirectory `
		-Filenames @("nvapi64.dll")
	
	# Copy the DLL files for NVENC to System32
	Write-Host "`nEnabling NVIDIA NVENC support:"
	CopyToSystem32 `
		-SourceDirectory $nvidiaDirectory `
		-Filenames @("nvEncodeAPI64.dll", "nvEncMFTH264x.dll", "nvEncMFThevcx.dll")
	
	# Copy the DLL files for NVDEC (formerly known as CUVID) to System32
	Write-Host "`nEnabling NVIDIA CUVID/NVDEC support:"
	CopyToSystem32 `
		-SourceDirectory $nvidiaDirectory `
		-Filenames @("nvcuvid64.dll", "nvDecMFTMjpeg.dll", "nvDecMFTMjpegx.dll") `
		-Rename @{"nvcuvid64.dll" = "nvcuvid.dll"}
	
	# Copy the DLL files for CUDA to System32
	Write-Host "`nEnabling NVIDIA CUDA support:"
	CopyToSystem32 `
		-SourceDirectory $nvidiaDirectory `
		-Filenames @("nvcuda64.dll", "nvcuda_loader64.dll", "nvptxJitCompiler64.dll") `
		-Rename @{"nvcuda_loader64.dll" = "nvcuda.dll"}
	
	# Print a blank line before any subsequent output
	Write-Host ""
}
{% endhighlight %}

This will locate and copy the necessary files in order to enable the following APIs:

- [NVAPI](https://developer.nvidia.com/nvapi): provides low-level access to NVIDIA GPUs and drivers. This API is used by the Unreal Engine for optimising performance on NVIDIA GPUs under Windows.

- [NVENC](https://developer.nvidia.com/nvidia-video-codec-sdk#NVENCFeatures) and [NVDEC](https://developer.nvidia.com/nvidia-video-codec-sdk#NVDECFeatures): provides hardware accelerated video encoding and decoding functionality. This API is used by the Unreal Engine for Pixel Streaming.

- [CUDA](https://developer.nvidia.com/cuda-zone): provides GPGPU compute functionality. This API is not currently used by the Unreal Engine under Windows at the time of writing, but included here due to its widespread popularity amongst developers.


## Future support

Microsoft has stated on multiple occasions that they been investigating support for non-DirectX graphics APIs in GPU accelerated Windows containers ever since the [original public announcement](https://techcommunity.microsoft.com/t5/containers/bringing-gpu-acceleration-to-windows-containers/ba-p/393939) in 2019. The [Container support for non-DX APIs](https://docs.microsoft.com/en-us/windows-hardware/drivers/display/container-non-dx) page of the Windows Driver Documentation details some of the work that has been done to assist hardware vendors in preparing their graphics drivers for expanded use inside Windows containers. Once the automatic file copy feature described on that page is supported for process-isolated Windows containers in addition to GPU-PV, it will no longer be necessary to manually copy files at container startup using a helper script as shown in this blog post. The set of supported graphics APIs will be determined by the registry entries that hardware vendors ship with their graphics drivers, and it is reasonable to expect that these registry entries will be expanded to encompass a broader set of vendor-specific APIs as further testing and verification is performed.

Until official support is provided by Microsoft and the graphics hardware vendors, we encourage developers who are interested in using vendor-specific graphics APIs inside GPU accelerated Windows containers to experiment with the approach described by this blog post and to communicate with Microsoft about the use cases that benefit from this expanded API support so that they can better understand the community's requirements and how to address them.
