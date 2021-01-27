---
layout: post
title: Investigating Unreal Engine performance in Linux cloud environments
author: Luke Bermingham, Sarah Krivan, Nicholas Pace, Aidan Possemiers and Adam Rehn
updated: 2021-01-27
tagline: An empirical investigation into the performance characteristics of Unreal Engine workloads running in Linux cloud environments.
---

{% capture _alert_content %}
1. Part one (this post) examines performance characteristics in Linux cloud environments.
2. Part two (coming later) examines performance characteristics in Windows cloud environments.
{% endcapture %}
{% include alerts/post-series.html content=_alert_content %}


## Overview
{:.no_toc}

The purpose of this investigation is to explore the performance characteristics of Unreal Engine workloads running in Linux cloud environments and to quantify the performance implications of utilising Docker containers and the Kubernetes container orchestration framework. For the purposes of this investigation we use Amazon AWS as the cloud platform for our experiments.

This investigation was performed by Dr Luke Bermingham, Dr Sarah Krivan, Nicholas Pace, Aidan Possemiers and Dr Adam Rehn from [TensorWorks](https://tensorworks.com.au).

***Special thanks to both Epic Games and NVIDIA for providing guidance and feedback during this investigation.***


## Contents
{:.no_toc}

* TOC
{:toc}


## 1. Methodology

To ensure sufficient reproducibility for our experiments we created a series of scripts to automate the deployment, execution, and performance profiling of a given Unreal Engine workload across several virtualised cloud configurations. After preliminary testing we selected [Epic Games' Infiltrator demo](https://www.unrealengine.com/marketplace/en-US/product/infiltrator-demo) as our workload due to its graphical complexity, packaged using Unreal Engine 4.25.3.

To identify the relative performance characteristics of containerised and non-containerised workloads our automation scripts deploy the Infiltrator demo to three specific cloud configurations:

- Cloud VM with no containerisation (EC2)
- Docker container running directly in a cloud VM (EC2)
- Container running in a managed Kubernetes service (EKS)

The underlying VMs for all three configurations use the same publicly available [g4dn.xlarge](https://aws.amazon.com/ec2/instance-types/g4/) instance type with an NVIDIA T4 GPU, the latest AWS-provided NVIDIA GRID drivers (450.80.02) and run the same *"Ubuntu 18.04 EKS K8s 1.16 - 2020.07.29"* Amazon Machine Image (AMI), which is a performance-optimised image specifically designed for running containerised workloads. This consistency minimises performance variations associated with the underlying hardware or operating system.

For each configuration, we run the Infiltrator demo 10 times with the Unreal Engine's CSV profiler enabled to capture a series of metrics including frame time, memory usage, GPU time, and game thread time. In order to control for performance deviations on a run-to-run basis we destroy and re-create the computing resources between each run to minimise caching effects. All rendering is performed offscreen using the Vulkan backend at a resolution of 1280x720 pixels.


## 2. Experimental results

Using the experimental methodology described in the section above, we collected the following metrics when running the Infiltrator demo across all three cloud configurations:

- Frame time
- Game thread time
- GPU time
- Total available memory

For the sake of brevity in the sections that follow, we refer to the three configurations as **"Cloud VM"** *(Cloud VM with no containerisation)*, **"Cloud Container"** *(Docker container running directly in a cloud VM)*, and **"Cloud Kube"** *(Container running in a managed Kubernetes service)*, respectively. The "Cloud VM" configuration is treated as the baseline against which the performance characteristics of the two containerised configurations are compared.

### 2.1. Data cleaning and Preprocessing

To reduce the effects of extreme outliers in the data we performed preliminary data cleaning and preprocessing of the collected samples prior to statistical analysis:

- All performance metrics exhibited significantly different characteristics during the initial startup of the Unreal Engine workload, ostensibly due to asset and level loading overheads. We discarded the first 10 samples collected from each run in order to eliminate this startup phase and focus our analysis purely on the performance characteristics of the Unreal Engine once steady-state has been achieved.

- To eliminate extreme outliers, we removed all samples that fell beyond three standard deviations from the mean of their given dataset. This preprocessing removed approximately 0.02% of samples on average, ensuring that the overwhelming majority of the data collected was still used during our statistical analysis whilst avoiding skewing effects.

### 2.2. Frame Time

Using the CSV profiler we collected the time taken in milliseconds to render each frame of the Infiltrator demo in each of the three cloud configurations. The mean, median, and standard deviation frame time values are listed in Table 1. The collected frame time samples are plotted in a box plot depicted by Figure 1. Outliers remaining after preprocessing were retained for statistical analysis, but removed from the box plot for ease of interpretation.

{% capture _table %}

|Metric                  |Cloud VM |Cloud Container |Cloud Kube |
|------------------------|---------|----------------|-----------|
|Mean (ms)               |10.02    |10.11           |10.36      |
|Median (ms)             |8.79     |8.86            |9.06       |
|Standard Deviation (ms) |4.66     |4.70            |4.81       |

{% endcapture %}
{% include tables/markdown.html table=_table caption="Table 1: Frame time (ms) running the Infiltrator demo in various Linux cloud configurations." %}

{% include figures/post.html image="figure1.svg" caption="Figure 1: Variation of frame times (ms) running the Infiltrator demo in various Linux cloud configurations (lower is better.)" %}

### 2.3. Game Thread Time

Using the CSV profiler we collected the time spent in milliseconds on the game thread for each frame of the Infiltrator demo in each of the three cloud configurations. The mean, median, and standard deviation game thread time values are listed in Table 2. The collected game thread time samples are plotted in a box plot depicted by Figure 2. Outliers remaining after preprocessing were retained for statistical analysis, but removed from the box plot for ease of interpretation.

{% capture _table %}

|Metric                  |Cloud VM |Cloud Container |Cloud Kube |
|------------------------|---------|----------------|-----------|
|Mean (ms)               |7.43     |7.46            |7.60       |
|Median (ms)             |6.14     |6.14            |6.27       |
|Standard Deviation (ms) |4.03     |4.06            |4.12       |

{% endcapture %}
{% include tables/markdown.html table=_table caption="Table 2: Game thread time (ms) spent running the Infiltrator demo in various Linux cloud configurations (lower is better.)" %}

{% include figures/post.html image="figure2.svg" caption="Figure 2: Variation of time spent in the game thread (ms) when running the Infiltrator demo in various Linux cloud configurations (lower is better.)" %}

### 2.4. GPU Time

Using the CSV profiler we collected the time spent in milliseconds on the GPU for each frame of the Infiltrator demo in each of the three cloud configurations. The mean, median, and standard deviation GPU time values are listed in Table 3. The collected GPU time samples are plotted in a box plot depicted by Figure 3. Outliers remaining after preprocessing were retained for statistical analysis, but removed from the box plot for ease of interpretation.

{% capture _table %}

|Metric                  |Cloud VM |Cloud Container |Cloud Kube |
|------------------------|---------|----------------|-----------|
|Mean (ms)               |8.60     |8.67            |8.77       |
|Median (ms)             |8.07     |8.13            |8.22       |
|Standard Deviation (ms) |3.23     |3.28            |3.34       |

{% endcapture %}
{% include tables/markdown.html table=_table caption="Table 3: GPU time (ms) spent running the Infiltrator demo in various Linux cloud configurations." %}

{% include figures/post.html image="figure3.svg" caption="Figure 3: Variation of time spent on the GPU (ms) when running the Infiltrator demo in various Linux cloud configurations (lower is better.)" %}

### 2.5. Memory

Using the CSV profiler we collected the total available free memory at each frame of the Infiltrator demo in each of the three cloud configurations. In addition to computing the mean, median, and standard deviation for analysis we also extracted the median value at the bottom 10th percentile to inform us about the peak memory consumption in each cloud configuration. These statistics are listed in Table 4 and accompanied with a plot of the minimum free memory values for each cloud configuration depicted in Figure 4.

{% capture _table %}

|Metric                         |Cloud VM |Cloud Container |Cloud Kube |
|-------------------------------|---------|----------------|-----------|
|Mean (MB)                      |12771    |12749           |12634      |
|Median (MB)                    |12773    |12753           |12636      |
|Standard Deviation (MB)        |34.88    |32.01           |33.33      |
|Median of 10th percentile (MB) |12726    |12703           |12591      |

{% endcapture %}
{% include tables/markdown.html table=_table caption="Table 4: Free memory (MB) available when running the Infiltrator demo in various Linux cloud configurations." %}

{% include figures/post.html image="figure4.svg" caption="Figure 4: Minimum free memory (MB) available when running the Infiltrator demo in various Linux cloud configurations (higher is better.)" %}


## 3. Discussion

Analysis of the frame time data in Table 1 and Figure 1 reveals that there is a small performance overhead when running Unreal Engine workloads in a container. The Infiltrator demo running in the "Cloud VM" configuration achieved the best median and average frametime. Using this as our baseline, we can deduce that running this workload in a container produced a frame time overhead of approximately 0.09ms (0.9%) on average or 0.07ms (0.8%) in the median case. In the case of Kubernetes, this frame time overhead was further increased to 0.34ms (3.3%) on average or 0.27ms (3.0%) in the median case.

Further analysis of the results shows that the same trend is repeated when measuring game thread time, GPU time, and available memory. That is, a slight overhead is introduced when running an Unreal Engine workload in a container and this overhead is increased further when running the workload on a Kubernetes worker node. Furthermore, when we analyse Table 4 we observe that the workloads running in Kubernetes have less available memory than their non-Kubernetes counterparts. The quantity is fairly negligible, reaching at worst approximately 140MB less memory than the Cloud VM configuration. This is to be expected, given that a workload running in a Kubernetes environment must share its resources with other Kubernetes processes running on the worker node such as the Kubelet agent.

Minor performance overheads aside, analysis of the frame times listed in Table 1 shows that all containerised configurations of the Infiltrator demo ran at approximately 110fps in the median case. This demonstrates the viability of running interactive Unreal Engine workloads such as games and Pixel Streaming applications in both simple containers and in Kubernetes clusters. **It is worth noting that these excellent results were only achieved when using correctly optimised cloud environments.** During the initial development of our experiments, we performed tests using an AMI which was not optimised for container workloads and installed the standard NVIDIA 440.0 gaming drivers instead of the newer AWS-provided NVIDIA GRID drivers. Under these conditions, performance overheads were significantly higher for both the containerised and Kubernetes configurations, and the Kubernetes configuration in particular suffered from noticeable performance variations. Performance improved significantly across all configurations when we switched to the AWS-provided NVIDIA GRID 450.80.02 drivers, and the remaining performance deltas for the containerised and Kubernetes configurations improved notably when we switched to an AMI that is optimised for container workloads.


## 4. Limitations

There are two key limitations that should be noted when considering the results of our experiments:

- A Development build of the Infiltrator demo was used due to the inability to run the CSV Profiler in Shipping builds. Shipping builds would likely exhibit greater performance and reduced memory use due to the absence of checks and developer tooling which are present in Development builds.

- The overhead of the CSV Profiler itself is inherently included in the profiling results and we expect that workloads running without the CSV Profiler would exhibit marginally greater performance.

Although both of these factors limit the total performance that can be achieved on any given hardware, the performance implications are consistent across all tested cloud configurations. As such, these limitations are only of consequence when considering absolute performance results and do not impact the conclusions drawn on the basis of relative performance deltas.


## 5. Summary and recommendations

The key findings of this investigation can be summarised as follows:

- The use of containerisation introduces extremely minor performance and resource usage overheads across CPU, GPU and system memory when running Unreal Engine workloads in cloud environments. Average frame time increases are in the order of less than half a millisecond.

- Experiments running the Infiltrator demo across all containerised and non-containerised configurations exhibited median frame times of approximately 110fps. This level of performance was achieved whilst actively profiling a Development build of the project. We expect that performance would be greater in a realistic deployment scenario where optimised Shipping builds of projects are running without the overheads of instrumentation.

Based on these findings, it is our conclusion that containerised cloud environments provide sufficient performance characteristics for the deployment and operation of interactive Unreal Engine workloads. For the best performance we strongly recommend using the latest GPU drivers prescribed for the particular instance of cloud compute being used (e.g. in the case of AWS G4 instances, the AWS provided GRID/Gaming drivers.) Additionally, if possible, we recommend using optimised or lightweight OS distributions or AMIs to minimise the likelihood of competition for resources as it seems containerised Unreal Engine workloads are particularly sensitive to experiencing performance variations in the presence of extraneous workloads on the host system.
