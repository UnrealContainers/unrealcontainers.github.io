---
title: Unreal Engine EULA Restrictions
tagline: Understand how the Unreal Engine EULA restricts the distribution of Unreal Engine container images.
order: 1
---

{% include alerts/warning.html title="The information provided on this page does not constitute legal advice." content="The authors of the Unreal Containers documentation are not lawyers and the information presented here is provided as general guidance only. For questions regarding your specific circumstances we strongly recommend that you seek professional legal advice." %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Preface

The [Unreal Engine End User License Agreement (EULA)](https://www.unrealengine.com/eula) governs the manner in which Engine licensees are permitted to access and distribute the Unreal Engine and its constituent components. It is important that all Engine licensees understand the entirety of this agreement before they start using the Unreal Engine, whether inside Docker containers or otherwise. This page assumes that the reader is already familiar with the EULA and its implications for using the Unreal Engine outside of containers, and highlights only the sections of the agreement that are directly relevant to the distribution of Unreal Engine container images.

Section 1A of the EULA enumerates the ways in which the Unreal Engine can be distributed. At the time of writing, section 1A contains six list items labelled A through F. List items A, B, C and D are the most relevant when considering the distribution of Unreal Engine container images. These list items are discussed in the sections below, followed by a summary providing a handy overview of the distribution mechanisms that are permitted or forbidden for the most commonly encountered distribution scenarios.


## EULA Section 1A, List Item A

List Item A is as follows:

> a. Distribution to end users - You may Distribute the Licensed Technology incorporated in object code format only as an inseparable part of a Product to end users who are subject to an end user license agreement which explicitly disclaims any representations, warranties, conditions, and liabilities related to the Licensed Technology. The Product may not contain any Paid Content Distributed in uncooked source format or any Engine Tools.

This list item is concerned with distribution of the Unreal Engine to end users, rather than other Engine licensees. When applied to container images, this simply means that a container image which includes a packaged Unreal project is subject to the same distribution terms as any packaged Unreal project would be when distributed outside of a container image.

It is worth noting, however, that distribution of "Engine Tools" to end users is not permitted. Engine Tools are formally defined in Section 25 of the EULA:

> "Engine Tools" means (a) editors and other tools included in the Engine Code; (b) any code and modules in either the Developer or Editor folders, including in object code format, whether statically or dynamically linked; and (c) other software that may be used to develop standalone products based on the Licensed Technology.

In a non-container context, this means that you can't bundle the build tools that you used to package your project alongside the packaged project itself. This restriction applies equally to container images: **you cannot distribute a container image which includes Engine Tools to end users.** This is important in a continuous integration and deployment context because it means you must copy the packaged project into a separate container image for distribution rather than leave it in an image derived from the container that was used to package the project. (The prohibitive size of the Engine Tools already makes this a practical necessity, but the EULA makes it a legal requirement as well.) Fortunately, [Docker multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/) make it easy to copy build artifacts into clean container images at the end of the build process, so compliance with this requirement is extremely simple to implement.



## EULA Section 1A, List Item B

List Item B is as follows:

> b. Distribution to other licensees - You may Distribute Engine Code (including as modified by you under the License) in Source Code or object code format, or any Content, to an Engine Licensee who has rights under its license to the same Version of the Engine Code or Content that you are Distributing.
> 
> Any public Distribution (i.e., intended for Engine Licensees generally) which includes Engine Tools (including as modified by you under the License) must take place either through the Marketplace (e.g., for distributing a Product’s modding tool or editor to end users) or through a fork of Epic’s GitHub UnrealEngine Network (e.g., for distributing source code). 

This list item is concerned with distribution of the Unreal Engine to other Engine licensees, rather than end users. The second half of the list item is extremely crucial in a container context, as it **forbids licensees from distributing container images which include Engine Tools via publicly-accessible container registries, [even if access to those registries is linked to an Epic Games account](https://github.com/adamrehn/ue4-docker/issues/16).** This means that any container images containing Engine Tools (e.g. images used for continuous integration purposes) can only be stored and shared via private container registries or other private sharing mechanisms.

It is worth noting that this restriction does not apply to container images which include only a packaged Unreal project without the Engine Tools. These container images can be shared freely under the terms set out in [List Item A](#eula-section-1a-list-item-a).


## EULA Section 1A, List Items C and D

List Item C is as follows:

> c. Distributions to employees and contractors - You also may Distribute Content (other than Paid Plug-ins) to an Engine Licensee who is your employee or your contractor who does not have rights under their license to the same Content, but only to permit that Engine Licensee to utilize that Content in good faith to develop a Product on your behalf for Distribution by you under the License, and not for the purpose of Content pooling or any other Distribution or sublicensing of Content that is not permitted under this Agreement. Recipients of such a Distribution have a limited license to use, reproduce, display, perform, and modify that Content to develop your Product as outlined above, and for no other purpose.

List Item D is as follows:

> d. Distribution of Paid Plug-ins - You may Distribute Paid Plug-ins to each of your Paid Plug-in Users so that they may use those Paid Plug-ins on your behalf under the License.

These list items extend the distribution rights described in [List Item B](#eula-section-1a-list-item-b) to employees and contractors who are working on the specific product for which the code or content in question is being distributed. It is important to note that this extension **does not apply to paid plugins**, which can only be distributed to your paid plugin users.


## Summary

Looking for a quick guide to whether a given distribution mechanism is permitted or forbidden for a particular scenario? Check out the handy lists below.

### Container images containing only packaged Unreal projects, with no source code, uncooked content, or Engine Tools

{% include alerts/info.html title="In a nutshell:" content="These images can be distributed via any mechanism so long as they comply with the disclaimer requirements of [List Item A](#eula-section-1a-list-item-a)." %}

You can distribute the images via these mechanisms:

{% capture _list %}
- Public repositories on container registries such as Docker Hub
- Private container repositories or container registries shared with Engine licensees inside your organisation or with your contractors
- Embedded in publicly-available virtual machine images (e.g. AMIs on the AWS Marketplace, VM images on the Azure marketplace, etc.)
- Embedded in private virtual machine images shared with Engine licensees inside your organisation or with your contractors
- Tarballs or other snapshots made publicly available for download
- Tarballs or other snapshots shared privately with Engine licensees inside your organisation or with your contractors
{% endcapture %}
{% include styled-lists/checks.html list=_list %}

### Container images containing Engine Tools or uncooked paid content, with no source code for paid plugins

{% include alerts/info.html title="In a nutshell:" content="These images can only be distributed privately within your organisation or with your contractors." %}

You can distribute the images via these mechanisms:

{% capture _list %}
- Private container repositories or container registries shared with Engine licensees inside your organisation or with your contractors
- Embedded in private virtual machine images shared with Engine licensees inside your organisation or with your contractors
- Tarballs or other snapshots shared privately with Engine licensees inside your organisation or with your contractors
{% endcapture %}
{% include styled-lists/checks.html list=_list %}

You **cannot** distribute the images via these mechanisms:

{% capture _list %}
- Public repositories on container registries such as Docker Hub
- Embedded in publicly-available virtual machine images (e.g. AMIs on the AWS Marketplace, VM images on the Azure marketplace, etc.)
- Tarballs or other snapshots made publicly available for download
{% endcapture %}
{% include styled-lists/crosses.html list=_list %}

### Container images containing source code for paid plugins

{% include alerts/info.html title="In a nutshell:" content="These images can only be distributed to your paid plugin users, irrespective of the mechanism." %}

You can distribute the images via these mechanisms:

{% capture _list %}
- Private container repositories or container registries shared with your paid plugin users only
- Embedded in private virtual machine images shared with your paid plugin users only
- Tarballs or other snapshots shared with your paid plugin users only
{% endcapture %}
{% include styled-lists/checks.html list=_list %}

You **cannot** distribute the images via these mechanisms:

{% capture _list %}
- Public repositories on container registries such as Docker Hub
- Private container repositories or container registries shared with Engine licensees inside your organisation or contractors who are not paid plugin users
- Embedded in publicly-available virtual machine images (e.g. AMIs on the AWS Marketplace, VM images on the Azure marketplace, etc.)
- Embedded in private virtual machine images shared with Engine licensees inside your organisation or contractors who are not paid plugin users
- Tarballs or other snapshots made publicly available for download
- Tarballs or other snapshots shared privately with Engine licensees inside your organisation or contractors who are not paid plugin users
{% endcapture %}
{% include styled-lists/crosses.html list=_list %}
