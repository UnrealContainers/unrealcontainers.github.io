---
title: Unreal Containers Community Hub
tagline: What is the Unreal Containers community hub and who maintains it?
order: 2
---

## Contents
{:.no_toc}

* TOC
{:toc}


## What is the Unreal Containers community hub?

Founded in 2019, the Unreal Containers community hub aims to act as a central collection of resources for assisting and connecting developers who wish to run Epic Games' [Unreal Engine](https://www.unrealengine.com/) inside [containers](https://www.docker.com/resources/what-container). In addition to providing comprehensive documentation and community-curated resources, the community hub also showcases examples of developers already deploying Unreal Engine containers for production workloads, and maintains a list of support providers who offer professional assistance in using Unreal Engine containers. The community hub is designed to address the following goals:

- To raise awareness amongst the Unreal Engine developer community of the capabilities that are offered by Unreal Engine containers.

- To provide comprehensive training resources to educate developers in the use of Unreal Engine containers across a variety of usage scenarios and container host environments.

- To connect developers who are interested in Unreal Engine containers with both their fellow developers and with support providers who can assist them in using Unreal Engine containers to achieve their goals.


## Who maintains the Unreal Containers community hub?

The Unreal Containers community hub was founded by [Dr Adam Rehn](https://adamrehn.com), a university lecturer, researcher, and director of the applied research company [TensorWorks](https://tensorworks.com.au). The community hub is primarily maintained by Dr Rehn, with additional assistance from community contributors. If you would like to contribute to the maintenance of the community hub then you can either visit the [Unreal Containers GitHub organisation]({{ site.data.nav.social[1].link }}) or engage the services of one of the [support providers]({{ "/support" | relative_uri }}) who provide financial sponsorhip to the community hub.


## Who sponsors the maintenance of the community hub?

The following organisations and initiatives provide, or have provided, financial sponsorhip to the Unreal Containers community hub:

{% for sponsor in site.data.sponsors %}
- [**{{ sponsor.name }}**]({{ sponsor.link }}): {{ sponsor.blurb }}
{% endfor %}
