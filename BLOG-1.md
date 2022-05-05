# All you need to know about the OpenJ9 JITServer

## What is OpenJ9?

`OpenJ9` is an open source JVM provided by the Eclipse Foundation, and has been available since 2017.

`OpenJ9` began life some 15 years ago as simply `J9`. Developed by IBM, `J9` has been used by all IBM Java based customers for all types of workloads - from medical, to banking and research.

As a major contributor to open source projects, IBM moved the development and governance of the `J9` JVM to the Eclipse Foundation, and re-branded the name to `OpenJ9`.

## How does OpenJ9 compare against other JVMs?

Optimized for the cloud and constrained environments, `OpenJ9` has the following advantages:

* Uses dramatically less memory without sacrificing application responsiveness.
* Provides AOT (Ahead of Time compilation) capabilities using a shared cache between instances.
* Provides an out-of-process JIT compiler (JITServer) to offload compilations to a remote server and eliminate CPU and memory spikes from the application instance.

![openj9-vs-hotspot](doc/source/images/openj9-vs-hotspot.png)

And, as with other open source JVMs, it is free of charge. Pay-for-support options are also available if needed.

## What is a JITServer?

The JITServer is a unique feature offered by the OpenJ9 JVM, and provides a big performance increase, especially when dealing with constrained environments such as micro-containers.

Instead of a typical container running a JVM with an internal JIT compiler, the JITServer is run remotely (on cloud or in its own container), providing JIT compilation services for multiple client JVMs. This approach has the following advantages:

* JIT compilation resources can be scaled independently from the Java application resources.
* Application containers can use smaller memory limits to minimize costs.
* Overall cluster memory utilization (JITServer included) is reduced, because memory consumption peaks from different applications don’t align.
* Provisioning is simpler - user only needs to care about the Java application requirements.
* Ramp-up is faster, especially in constrained environments.
* Performance of short-lived applications is better.
* Performance is more predictable - less CPU spikes in the JVM.
* Autoscaling behavior is better (a direct consequence of faster ramp-up).

## How do I get it?

As mentioned previously, the `OpenJ9 JVM` is open source and managed by the [Eclipse Foundation](https://www.eclipse.org/openj9/). The `JITServer` technology is included with the `OpenJ9 JVM` - in reality, the `JITServer` is just another persona instance of the `OpenJ9 JVM`.

Pre-built `OpenJ9 JVM` binaries are available for free from IBM, as part of the [IBM Semeru Runtimes](https://developer.ibm.com/languages/java/semeru-runtimes/downloads/). Note that the [AdoptOpenJDK](https://adoptopenjdk.net/) project, the former home for OpenJ9 binaries, has been deprecated and the link provided when selecting the `OpenJ9 JVM` will direct the user to the `IBM Semeru` download page.

Containers with OpenJ9 JVM can be pulled from [dockerhub](https://hub.docker.com/_/ibm-semeru-runtimes) or from [RedHat catalog](https://catalog.redhat.com/software/containers/search?q=semeru&p=1).


## Want to dig deeper?

Click here to take a deeper dive into the JITServer details.

## How about a demo?

Click here to check out a demo where we show how to configure and run multiple containers with a JITServer. We will also use Grafana to graph CPU and memory metrics showing how the JITServer can better utilize and minimize system resources.

![demo](doc/source/images/demo.png)

