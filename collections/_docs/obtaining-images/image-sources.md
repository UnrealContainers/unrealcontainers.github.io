---
title: Available Image Sources
tagline: What EULA-compliant sources are available for obtaining Unreal Engine container images?
order: 2
---

{% capture _prerequisites %}
- [Development images vs. runtime images](../concepts/image-types)
- [Unreal Engine EULA Restrictions](./eula-restrictions)
{% endcapture %}
{% include alerts/prerequisites.html pages=_prerequisites %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Sources of Unreal Engine runtime images

[Runtime images](../concepts/image-types#runtime-images) are container base images for running packaged Unreal projects. Packaged Unreal projects can run in any container image that includes the required runtime libraries. These requirements vary based on platform:

- **Windows containers:** the [mcr.microsoft.com/windows](https://hub.docker.com/_/microsoft-windows) image will work out of the box, but is prohibitively large in size and impractical to deploy. The [mcr.microsoft.com/windows/servercore](https://hub.docker.com/_/microsoft-windows-servercore) image is smaller but will require a number of DLL files copied from either the full base image or the host system. Don't even bother trying to use the [mcr.microsoft.com/windows/nanoserver](https://hub.docker.com/_/microsoft-windows-nanoserver) image.

- **Linux containers:** any image based on a full Linux distribution should work out of the box (e.g. [Fedora](https://hub.docker.com/_/fedora), [CentOS](https://hub.docker.com/_/centos), [Ubuntu](https://hub.docker.com/_/ubuntu), etc.) but minimal distributions like [Alpine](https://hub.docker.com/_/alpine) will most likely have issues.

- **GPU accelerated Linux containers with the NVIDIA Container Toolkit:** performing offscreen rendering inside a container requires an image derived from the [nvidia/opengl](https://hub.docker.com/r/nvidia/opengl/), [nvidia/vulkan](https://hub.docker.com/r/nvidia/vulkan/) or [nvidia/cudagl](https://hub.docker.com/r/nvidia/cudagl/) base images. Creating X11 windows also requires the X11 runtime libraries.

Since runtime base images do not contain any Unreal Engine components, they can be [distributed without any restrictions](./eula-restrictions) and shared publicly on container registries such as Docker Hub. A number of pre-configured base images are readily available for download:

- [Official Container Images](./official-images): the set of official Unreal Engine container images provided by Epic Games includes a number of runtime images for various use cases. The Linux runtime images are derived from the ue4-runtime repository listed immediately below.

- [adamrehn/ue4-runtime](https://hub.docker.com/r/adamrehn/ue4-runtime): provides a variety of tags representing minimal, pre-configured environments for running packaged Unreal Engine projects in GPU accelerated Linux containers via the [NVIDIA Container Toolkit](../concepts/nvidia-docker). Support for both OpenGL and Vulkan is provided by all image variants. Includes variants with CUDA support for use with Pixel Streaming applications or machine learning workloads.


## Sources of Unreal Engine development images

[Development images](../concepts/image-types#development-images) are images which contain the "Engine Tools" (Editor and build tools.) As discussed in the [Unreal Engine EULA Restrictions](./eula-restrictions) page, container images that include the Engine Tools for building and packaging Unreal projects or plugins cannot be distributed publicly. **These container images must only be distributed privately to ensure compliance with the terms of the EULA.**

Since building development images can be a time-consuming process, it is typically preferable to use pre-built images where available. If pre-built container images that meet your requirements are not available (or you wish to create container images for a custom version of the Unreal Engine) then you will need to build images and distribute them within your organisation via whichever private sharing mechanism is most appropriate.

**The following sources provide pre-built development container images**:

{% assign source-pages = site.documents | where: "category", page.category | where: "subcategory", page.subcategory | where_exp: "page", "page.order > 2" %}
{% for source in site.data.image-sources.sources %}
{% assign page = source-pages | where: "source", source['Source'] | first %}
{% if source['Pre-built images'] == 'Yes' %}
- [{{ source['Source'] | escape }}]({{ page.url | relative_url }}): {{ page.tagline | escape }}
{% endif %}
{% endfor %}

**The following sources are available for building development container images**:

{% for source in site.data.image-sources.sources %}
{% assign page = source-pages | where: "source", source['Source'] | first %}
{% if source['Pre-built images'] != 'Yes' %}
- [{{ source['Source'] | escape }}]({{ page.url | relative_url }}): {{ page.tagline | escape }}
{% endif %}
{% endfor %}

For details of the features supported by these sources, see the section below.


## Feature support matrix

The table below lists the features that are supported by each source of development container images:

{% assign cols = site.data.image-sources.sources.size | plus: 1 %}
{::nomarkdown}
<figure>
	<table>
		<thead>
			<tr>
				<th rowspan="2">Feature</th>
				<th colspan="{{ cols }}" class="text-center">Source</th>
			</tr>
			<tr>
				{% for source in site.data.image-sources.sources %}
					<th><a href="{{ source-pages | where: 'source', source['Source'] | map: 'url' | first | relative_url }}">{{ source['Source'] | escape }}</a></th>
				{% endfor %}
			</tr>
			<tbody>
				{% for category in site.data.image-sources.features %}
					<tr><th colspan="{{ cols }}">{{ category[0] | escape }}</th></tr>
					{% for feature in category[1] %}
						<tr>
							<td>{{ feature.name | escape }}</td>
							
							{% for source in site.data.image-sources.sources %}
								{% assign value = source[ feature.name ] %}
								{% case value %}
									{% when 'Yes' %}
										{% assign class = 'supported' %}
									{% when 'No' %}
										{% assign class = 'unsupported' %}
									{% else %}
										{% assign class = '' %}
								{% endcase %}
								
								<td class="{{ class }}">{{ value | escape }}</td>
							{% endfor %}
						</tr>
					{% endfor %}
				{% endfor %}
			</tbody>
		</thead>
	</table>
</figure>
{:/}

For an explanation of what each feature listed in the table denotes, see below.

{% for category in site.data.image-sources.features %}
#### {{ category[0] | escape }}
{:.no_toc}

{% for feature in category[1] %}
- **{{ feature.name | escape }}:** {{ feature.description }}
{% endfor %}
{% endfor %}
