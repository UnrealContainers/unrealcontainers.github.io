---
title: Linux Installed Builds
tagline: Use container images as a source of pre-built binaries in lieu of the Epic Games Launcher under Linux.
quickstart: "workflows"
order: 8
---

{% capture _alert_content %}
- An [Unreal Engine development image](../concepts/image-types) that includes a Linux Installed Build of the Engine Tools
- A Linux environment [configured for running containers](../environments) (this is just to extract files from the container images)

{% endcapture %}
{% include alerts/required.html content=_alert_content %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Overview

Under Windows and macOS, the Epic Games Launcher provides pre-built Engine binaries for download by all Engine Licensees. However, the lack of a native Linux version of the Launcher leaves Linux users without a readily available source of pre-built binaries. This complicates the onboarding process for Linux-based developers due to the need to compile the Engine from source. Fortunately, Unreal Engine container images can act as a source of pre-built Engine binaries that can be copied from the container filesystem to the host and then distributed via an organisation's internal distribution mechanisms.


## Key considerations

- This process is far better suited to [Installed Builds](https://docs.unrealengine.com/en-us/Programming/Deployment/UsinganInstalledBuild) of the Engine rather than source builds, since Installed Builds are specifically designed to be portable.

- Unreal Engine version 4.21.0 is the first version that is completely self-contained under Linux thanks to the use of bundled versions of both Mono and the compiler toolchain. Earlier versions of the Engine will require an accompanying system installation of clang and/or Mono on any machines to which they are deployed.

- If the Unreal Engine installation inside a given container image doesn't contain any [symbolic links](https://en.wikipedia.org/wiki/Symbolic_link) then the directory structure can simply be copied from the container filesystem to the host filesystem without any further modifications. However, if symlinks are used then additional configuration may be required to update the symlink targets to paths that are appropriate for the host filesystem. Typically, any [source of development container images](../obtaining-images/image-sources#sources-of-unreal-engine-development-images) that relies on symlinks will provide export functionality that performs this configuration for you automatically.


## Implementation guidelines

### Using container images from the ue4-docker project

The [ue4-docker project](../obtaining-images/ue4-docker) provides the [ue4-docker export](https://docs.adamrehn.com/ue4-docker/commands/export) command to export Installed Builds from the [ue4-full container image](https://docs.adamrehn.com/ue4-docker/building-images/available-container-images#ue4-full). This command works with container images for Unreal Engine 4.21.0 and newer, and performs symlink retargeting on the host system automatically. Due to the presence of symlinks in the container images built by ue4-docker, manually copying the files from the container filesystem to the host filesystem is not recommended.

### Using container images built from custom Dockerfiles

You can use the [docker cp](https://docs.docker.com/engine/reference/commandline/cp/) command to copy the directory structure of an Unreal Engine installation from the filesystem of a running container to the host filesystem. For example, we could run the following commands to perform an export if our container image is tagged `unreal-engine:latest` and the root of the Unreal Engine installation is located at `/home/ue4/UnrealEngine` within the container filesystem:

{% highlight bash %}
# Start a container and leave it running in the background
docker run --rm -d --name ue4 "unreal-engine:latest" bash -c "sleep infinity"

# Copy the Unreal Engine installation from the container filesystem to the host filesystem
docker cp "ue4:/home/ue4/UnrealEngine" "/destination/path/on/host/filesystem"

# Stop the container
docker stop ue4
{% endhighlight %}

If the directory structure of the Unreal Engine installation contains any symlinks then they will need to be removed at this point and replaced with symlinks to the appropriate paths on the host filesystem. When [writing your own Dockerfiles](../obtaining-images/write-your-own) it is good practice to either document these symlinks or provide an automated method for modifying them.


## Related media

{% include related-items/media.html url=page.url %}
