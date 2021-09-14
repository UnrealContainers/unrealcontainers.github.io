---
title: Official Container Images
tagline: Use the official Unreal Engine container images provided by Epic Games.
source: Official Container Images
order: 3
---

{% include alerts/source-features.html source=page.source %}

{% include alerts/warning.html title="Pre-built images coming soon!" content="Although the [Unreal Engine documentation](https://docs.unrealengine.com/4.27/en-US/SharingAndReleasing/Containers/) makes reference to pre-built container images on GitHub Container Registry, these images are not yet available. These images are expected to be available soon." %}


## About this container image source

In April 2021, Epic Games [announced on the Unreal Engine public roadmap](https://portal.productboard.com/epicgames/1-unreal-engine-public-roadmap/c/320-ue-container-build-beta) the intention to include official support for containers in Unreal Engine 4.27. This support subsequently [landed in Unreal Engine 4.27.0](https://docs.unrealengine.com/4.27/en-US/WhatsNew/Builds/ReleaseNotes/4_27/#containerdeployment_beta_) in August 2021. This version and all subsequent versions of the Unreal Engine ship with a set of official container images which includes both [development images](../concepts/image-types#development-images) and [runtime images](../concepts/image-types#runtime-images), along with a number of supporting container images for other components related to various Unreal Engine use cases:

- The development container images that ship with the Unreal Engine are derived from the [ue4-docker project](./ue4-docker). The standalone Dockerfiles for these images are generated using the export and combination functionality described in the blog post [Preview the future of the ue4-docker project](https://adamrehn.com/articles/preview-the-future-of-ue4-docker/).

- The Linux runtime container images that ship with the Unreal Engine are derived from the [ue4-runtime project](https://github.com/adamrehn/ue4-runtime). A Windows runtime container image is also provided based on the code presented in the blog posts [Offscreen rendering in Windows containers](../../blog/offscreen-rendering-in-windows-containers/) and [Enabling vendor-specific graphics APIs in Windows containers](../../blog/enabling-vendor-specific-graphics-apis-in-windows-containers/).

The source code for all of the official container images can be found under the [Engine/Extras/Containers/Dockerfiles](https://github.com/EpicGames/UnrealEngine/tree/release/Engine/Extras/Containers/Dockerfiles/) directory of the Unreal Engine source tree. **Note that if you have downloaded the Unreal Engine source code from GitHub then you will need to run `Setup.bat` (under Windows) or `Setup.sh` (under other platforms) to retrieve the source files, since these files are currently treated as binary dependencies and are not included in the git repository itself.**

The code for the official container images [was contributed to the Unreal Engine by Dr Adam Rehn](https://github.com/EpicGames/UnrealEngine/commits?author=adamrehn), the creator of the ue4-docker and ue4-runtime projects and the founder of the [Unreal Containers community hub]({{ "/" | relative_url }}).

The official Unreal Engine documentation features a [section covering official container support](https://docs.unrealengine.com/4.27/en-US/SharingAndReleasing/Containers/). The documentation includes an overview of the container images that ship with the Unreal Engine, getting started guides, and how-to guides for specific tasks. To avoid unnecessary duplication of information, the Unreal Engine documentation primarily provides details specific to the official container images and links to the Unreal Containers community hub to provide general context.


## Using this container image source

- Start by reading the [Containers Overview](https://docs.unrealengine.com/4.27/en-US/SharingAndReleasing/Containers/ContainersOverview/) page of the Unreal Engine documentation, which provides an overview of container support in the Unreal Engine and lists the official container images that ship with Unreal Engine 4.27 and newer.

- Work through the [Containers Quick Start](https://docs.unrealengine.com/4.27/en-US/SharingAndReleasing/Containers/ContainersQuickStart/) guide to pull a pre-built Linux development container image and use it to package an Unreal project.

- If you want to build the official images from source (e.g. for a custom version of the Unreal Engine) then check out the [How To Guides](https://docs.unrealengine.com/4.27/en-US/SharingAndReleasing/Containers/HowTo/) for step-by-step instructions.
