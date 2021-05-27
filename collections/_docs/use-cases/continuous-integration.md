---
title: Continuous Integration and Deployment (CI/CD)
tagline: Use reproducible environments to build and deploy Unreal Engine projects and plugins.
quickstart: "build"
order: 1
---

{% capture _alert_content %}
- Unreal Engine container images that [include the Engine Tools and support building for the desired target platform(s)](../obtaining-images/image-sources)
- Environments [configured for running the containers](../environments) for each applicable host platform
{% endcapture %}
{% include alerts/required.html content=_alert_content %}


## Contents
{:.no_toc}

* TOC
{:toc}


## Overview

One of the most common uses of Unreal Engine containers is to provide reproducible environments within which automated builds of Unreal projects and plugins can be performed. These environments can be used with any existing CI system that supports Docker containers, allowing developers to utilise their preferred infrastructure and leverage existing CI/CD workflows. These workflows can also be extended to produce container images that can then be used to run packaged Unreal projects in the cloud, facilitating other use cases such as [cloud rendering](./cloud-rendering) and building [microservices powered by the Unreal Engine](./microservices). The built container images can also be deployed and scaled with standard container orchestration technologies, allowing developers to comply with existing best practices for running containerised applications.


## Key considerations

- Methods of building and packaging Unreal projects that utilise a graphical interface will not work inside containers. If you are not already accustomed to building and packaging projects via the command-line then you should take the time to familiarise yourself with this process on a host system before attempting to perform packaging inside Docker containers.

- You will need to make sure your chosen CI system supports running build commands inside Docker containers. If you are using [Windows containers](../concepts/windows-containers) then you will also need to check that specific support for Windows containers is included, as some CI systems only include support for Linux containers.

- If you're building Docker images for deploying packaged projects then you will need to make sure your chosen CI system supports building container images from Dockerfiles.


## Implementation guidelines

### Using the tools that ship with the Engine

#### Building and packaging

The primary method of building and packaging Unreal projects and plugins via the command-line is using commands provided by the [Unreal Automation Tool (UAT)](https://docs.unrealengine.com/en-US/Programming/BuildTools/AutomationTool):

- The [`BuildCookRun` command](https://docs.unrealengine.com/en-US/Engine/Deployment/BuildOperations) is used for building and packaging projects.

- The `BuildPlugin` command is used for building and packaging plugins.

These commands can be invoked by using the `RunUAT` helper script, which is located under the `Engine/Build/BatchFiles` subdirectory of the Unreal Engine installation root. The same commands that are used to package projects and plugins on a host system can be used to package projects and plugins within any Unreal Engine container that includes the Engine Tools.

#### Copying packaged files

Once a project or plugin has been built and packaged there are a number of available mechanisms that can be used to copy the packaged files to the desired location:

- **Uploading files to remote storage.** File upload commands can be run directly inside Unreal Engine containers if desired. However, in most typical scenarios the CI system orchestrating the build will take responsibility for handling any permanent storage of build products.

- **Copying the files to bind-mounted host filesystem directories.** This is typically the most common approach, since the CI system orchestrating the build can then identify the build products on the host filesystem and take the appropriate actions to copy them to permanent storage locations.

- **Copying the files to a new container image for deployment.** If you are planning to deploy a packaged Unreal project using Docker containers then you can create the container images as part of the overarching build process. See the [Building container images for deployment](#building-container-images-for-deployment) section below for more details on this option.

#### Running tests

Prior to Unreal Engine version 4.21, the only testing mechanism exposed via the command-line was the [Automation System](https://docs.unrealengine.com/en-us/Programming/Automation), which can be accessed via the `automation` console command, either by invoking the Editor directly or via the `BuildCookRun` command provided by UAT. The command to run automation tests through the Editor directly looks like this:

{% highlight bash %}
# Starts the game, runs the specified automation tests, and then exits
# (Note the use of the `-nullrhi` flag to disable rendering, which allows this to work in containers without GPU access)
UE4Editor "/path/to/project.uproject" -game -buildmachine -stdout -unattended -nopause -nullrhi -nosplash -ExecCmds="automation RunTests <TEST1>+<TEST2>+<TESTN>;quit"
{% endhighlight %}

The command to run automation tests through UAT looks like this:

{% highlight bash %}
# Starts the game, runs the specified automation tests, and then exits
# (Note the use of the `-nullrhi` flag to disable rendering, which allows this to work in containers without GPU access)
RunUAT BuildCookRun -project="/path/to/project.uproject" -noP4 -buildmachine -unattended -nullrhi -run "-RunAutomationTest=<TEST1>+<TEST2>+<TESTN>"
{% endhighlight %}


Unreal Engine 4.21.0 introduced the Gauntlet Automation Framework, which allows developers to specify tests using C# scripts that are then run by the `RunUnreal` command provided by UAT. However, at the time of writing there is very little available documentation for Gauntlet and the authors of this documentation have not tested the use of Gauntlet within Unreal Engine containers.

### Using third-party infrastructure and abstractions

There are a number of infrastructure projects maintained by the community that abstract the process of building and packaging Unreal projects and plugins. These infrastructure projects typically automate the process of generating the appropriate arguments for the `BuildCookRun` command or `BuildPlugin` command and invoking UAT. Some example projects are listed below:

- [ue4cli](https://github.com/adamrehn/ue4cli): provides a command-line tool that automatically determines the correct path to the `RunUAT` helper script and generates the appropriate arguments for building and packaging the project or plugin in the current working directory. The [ue4-docker project](../obtaining-images/ue4-docker) includes ue4cli in the [ue4-full container image](https://docs.adamrehn.com/ue4-docker/building-images/available-container-images#ue4-full).

- [ue4-ci-helpers](https://github.com/adamrehn/ue4-ci-helpers): provides an API for scripting the build and packaging process for projects and plugins using Python code. Includes functionality for automatically producing compressed archives containing the packaged files. The [ue4-docker project](../obtaining-images/ue4-docker) includes ue4-ci-helpers in the [ue4-full container image](https://docs.adamrehn.com/ue4-docker/building-images/available-container-images#ue4-full).

In order to use any given infrastructure project it will need to be included in your container images. Unless your Unreal Engine container images already contain these components you will need to write a Dockerfile that extends an existing Unreal Engine container image and installs the desired infrastructure.

### Building container images for deployment

In a scenario where you're packaging a project for distribution via traditional mechanisms you will most likely start a container using an existing image and run a series of build commands inside that container. However, if you're deploying a packaged project using Docker containers, the entire build process can actually take place within a [docker build](https://docs.docker.com/engine/reference/commandline/build/) command that uses [Docker multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/) to copy the packaged files into a new container image at the end of the process. (In a dual-distribution scenario you can also copy the packaged files from the filesystem of the newly-built container image to the host filesystem and then deploy those files using traditional mechanisms.)

Writing a Dockerfile for a multi-stage build is quite straightforward:

- The container image that contains the Engine Tools and that will be used for building and packaging the project should be specified in the first `FROM` directive of the Dockerfile.

- This directive should be followed by one or more `RUN` directives to build and package the project using your preferred mechanisms.

- The Dockerfile should conclude with a second `FROM` directive that specifies the [base image that the packaged project will run inside](../obtaining-images/image-sources#sources-of-base-images-for-running-packaged-unreal-projects) and a `COPY` directive to copy the packaged files into the new image.

An example Dockerfile is provided below that builds and packages a project in the [ue4-full container image](https://docs.adamrehn.com/ue4-docker/building-images/available-container-images#ue4-full) from the [ue4-docker project](../obtaining-images/ue4-docker) and copies the packaged files into the [adamrehn/ue4-runtime](https://hub.docker.com/r/adamrehn/ue4-runtime) runtime image.

{% highlight docker %}
# Perform the build in an Unreal Engine container image that includes the Engine Tools
FROM adamrehn/ue4-full:4.21.2 AS builder

# Clone the source code for our Unreal project
RUN git clone --progress --depth 1 https://github.com/adamrehn/ue4-sample-project.git /tmp/project

# Build and package our Unreal project
# (We're using ue4cli for brevity here, but we could just as easily be calling RunUAT directly)
WORKDIR /tmp/project
RUN ue4 package

# Copy the packaged files into a container image that doesn't include any Unreal Engine components
FROM adamrehn/ue4-runtime:latest
COPY --from=builder --chown=ue4:ue4 /tmp/project/dist/LinuxNoEditor /home/ue4/project
{% endhighlight %}

### Using prebuilt Unreal plugins

Using prebuilt plugins can speed up a project's build process by avoiding the need to recompile all of the plugins the project depends on each time the project itself is rebuilt. There are two ways to obtain precompiled binaries for a plugin:

- **Unreal Marketplace plugins:** plugins distributed via the [Unreal Marketplace](https://www.unrealengine.com/marketplace/) typically include precompiled binaries for Windows, although not for Linux.

- **Packaging a plugin yourself:** if you have a custom plugin (or a Marketplace plugin that you want to build for Linux), you can build and package it yourself by running either the [ue4 package](https://docs.adamrehn.com/ue4cli/descriptor-commands/package) command from [ue4cli](https://github.com/adamrehn/ue4cli) or the underlying `BuildPlugin` script included with Unreal AutomationTool.

Once you have obtained precompiled binaries for a plugin, you can "install" the plugin by simply copying the files to the `Engine/Plugins` directory of your Unreal Engine installation. So long as you do not include a separate copy of the plugin in the `Plugins` directory of your Unreal project, the build process will use the precompiled binaries from the Engine. It's worth noting that some Marketplace plugins (such as the [Substance plugin](https://www.unrealengine.com/marketplace/en-US/product/substance-plugin)) include hard-coded checks to ensure they've been placed in a subdirectory consistent with a Marketplace installation, so you'll need to copy any such plugins into the `Engine/Plugins/Marketplace` directory of your Unreal Engine installation specifically. Fortunately, the overwhelming majority of plugins can be placed anywhere inside the `Engine/Plugins` directory and will work without complaint.

There are two common approaches for incorporating the precompiled binaries into a CI/CD pipeline:

- **Download the files from a remote server.** In this approach, the packaged plugin files are uploaded to a remote server or artifact management system and then downloaded as needed when building projects that consume them. This approach is extremely flexible because each plugin can be consumed by any number of projects, but requires additional infrastructure for storing and serving the files.

- **Bake the files into a container image.** In this approach, the packaged plugin files are incorporated into the container image that will be used as the build environment for a project. This approach is somewhat inflexible and can result in a large number of container images when you are building multiple projects depending on multiple plugins, but is suitable for simple projects and requires no additional infrastructure.


## Related media

{% include related-items/media.html url=page.url %}


## Related repositories

{% include related-items/repositories.html url=page.url %}
