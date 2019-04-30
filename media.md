---
layout: default
pageclass: media
title: Media
tagline: Videos and slide decks from live presentations and online training courses.
permalink: /media
---

## Contents
{:.no_toc}

* TOC
{:toc}


{% for category in site.data.media %}
## {{ category.name | escape }}

{% include lists/media.html items=category.items %}
{% endfor %}
