---
title: macOS Containers
tagline: Linux and Windows containers exist, but what about macOS?
quickstart: ["build"]
order: 3
---

{% capture _alert_content %}
- As of today, **there is no such thing as a macOS container.**
- There are alternatives available that use macOS virtual machines to approximate the experience of using containers.
- The open source community is working hard to change this situation and [bring native container support to macOS](https://macoscontainers.org).
{% endcapture %}
{% include alerts/overview.html content=_alert_content %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Container support on macOS (or lack thereof)

Container technology is supported by a wide range of operating systems, including [FreeBSD](https://en.wikipedia.org/wiki/FreeBSD_jail), [Solaris](https://en.wikipedia.org/wiki/Solaris_Containers), Linux and [Windows](./windows-containers). However, the macOS operating system does not feature native support for containers, and its underlying [XNU kernel](https://en.wikipedia.org/wiki/XNU) lacks a number of the isolation primitives required to implement container support with features equivalent to those found on other platforms. Apple has not indicated any intent to add first-party container support to macOS, nor have they made any public statement regarding their position on this topic. There is a great deal of community speculation regarding the underlying reasons for this, although the prevailing theory seems to be [a perceived lack of market demand](https://apple.stackexchange.com/a/294627).


## Container alternatives using macOS virtual machines

There are a number of open source and proprietary systems that enable the use of macOS virtual machines inside Linux containers, which can then be managed using existing container tooling:

- ***(Proprietary):*** [MacStadium Orka](https://www.macstadium.com/orka)
- ***(Open source):*** [Docker-OSX](https://github.com/sickcodes/Docker-OSX)
- ***(Open source):*** [sxkdvm](https://github.com/Cleafy/sxkdvm) *(appears to be no longer actively maintained)*


## The macOS Containers initiative

The [macOS Containers initiative](https://macoscontainers.org) is an effort by the open source development community to bring native container support to macOS without requiring any modifications to the operating system itself. The initiative is still at a very early stage and has not yet produced a working implementation.
