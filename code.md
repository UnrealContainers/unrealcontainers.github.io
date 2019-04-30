---
layout: default
pageclass: code
title: Code
tagline: Sample code repositories to demonstrate various use cases and library integrations.
permalink: /code
---


## Structure of this page
{:.no_toc}

The resources on this site are designed to be as implementation-agnostic as possible, providing information that applies to container images built using Dockerfiles from any source. However, any concrete example of code that works with Unreal Engine container images must, by necessity, target one or more [specific implementations](./docs/obtaining-images/image-sources). As such, the repositories listed on this page are grouped based on the container image implementations that they target, rather than the concepts that they demonstrate.


## Contents
{:.no_toc}

* TOC
{:toc}


{% for implementation in site.data.repositories %}
## Examples using container images built by {{ implementation.name | escape }}

{{ implementation.description }}

{% include lists/repositories.html repos=implementation.repos %}
{% endfor %}
