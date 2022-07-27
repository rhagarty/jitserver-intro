---
# Related publishing issue: https://github.ibm.com/IBMCode/IBMCodeContent/issues/7108 
# https://w3.ibm.com/developer/contentcreatortool/cct/developer/master/Tutorials/ 
draft: true
# ignore_prod: 		#Required=false (true|false) - If true, the changes to the content do not show on the live site. If false, the changes are published to the live site.
# display_in_listing: 		#Required=false (true|false) - If set to true (the default value), nothing happens. If set to false, the page will exist but it won't be displayed in hubs, archives, or search. Set this to false when you use the page_links_to metadata.

title: "Using OpenJ9 JITServer in Kubernetes"
subtitle: "For better JVM ramp-up, app autoscaling, QoS, smaller peak memory consumption, improved application density"
meta_title: "Using OpenJ9 JITServer in Kubernetes"

authors:
  - name: "Marius Pirvu"
    email: "mpirvu@ca.ibm.com"
  - name: "Rich Hagarty"
    email: "rich.hagarty@ibm.com"

completed_date: "2022-07-27"
# last_updated: "2022-07-27"
# archive_date: 		#Required=false - This date is when the content was archived. The date format is YYYY-MM-DD.
check_date: "2023-07-27"

time_to_read: 30 minutes

excerpt: "Authors demonstrate how to take advantage of the OpenJ9 JITServer technology in Kubernetes. Examples include deploying JITServer using YML files, deploying JITServer using the Helm chart, deploying a Sprint Boot application and connecting it to the JITServer compilation service, and verifying that the JITServer connection is successful."
meta_description: "Authors demonstrate how to take advantage of the OpenJ9 JITServer technology in Kubernetes. Examples include deploying JITServer using YML files, deploying JITServer using the Helm chart, deploying a Sprint Boot application and connecting it to the JITServer compilation service, and verifying that the JITServer connection is successful."
meta_keywords: "OpenJ9, JITServer, Kubernetes, pirvu"

# og_meta: 		#Required=false - Ignore this. Do not use this. Marketing team will specify complete set of og metadata and twitter metadata needed to display appropriate images
# twitter_meta: 		#Required=false - Ignore this. Do not use this. Marketing team will specify complete set of og metadata and twitter metadata needed to display appropriate images

primary_tag: "java"

tags: 
  - "semeru-runtimes"

components:
  - "kubernetes"
  - "open-j9"

# services: 		#Required=false - Select the 1 or more services that are specifically in use by the content. Less is more. (Services can only be the slugs from the https://github.ibm.com/IBMCode/Definitions/blob/master/services.yml file.) 
# runtimes: 		#Required=false - Select the 1 or more runtimes that are specifically in use by the content. Less is more. (Runtimes can only be the slugs from the https://github.ibm.com/IBMCode/Definitions/blob/master/runtimes.yml file.)

related_content: 
  - type: articles
    slug: garbage-collection-tradeoffs-and-tuning-with-openj9
  - type: articles
    slug: eclipse-openj9-class-sharing-in-docker-containers
  - type: articles
    slug: hands-on-with-openj9-benefits

# related_links: 		#Required=false - Specify 1 or more links to external content (that does not appear on IBM Developer). The description does not display of the link does NOT display on the page.
#  - title: 		#Required=true
#    url: 		#Required=true
#    description: 		#Required=false

# collections: 		#Required=false - Select the 1 or more collections that you want the content to appear in. (Collections can only be the slugs from the https://github.ibm.com/IBMCode/Definitions/blob/master/collections.yml file.)
# meta_tags: 		#Required=false - Ignore this. Do not use this. This is a comma separated list of tags used for SEO purposes. Only our SEO person will add these tags.
# social_media_meta: 		#Required=false - This is a description that goes in the social media description. Duplicate the meta_description.
# private_portals: 		#Required=false - Will this content be served directly to a private portal?
# content_tags: 		#Required=false - This is used to feature content on collections pages or on the community page. Only the editors of collections pages or the community page should use this metadata. Full list at https://github.ibm.com/IBMCode/Definitions/blob/master/content_tags.yml
# also_found_in: 		#Required=false - Specify the learning path, series, or collection that this content is included in. Use the following format: content-type/slug, such as 'learningpaths/get-started-with-deep-learning/'
# page_links_to: 		#Required=false - Provide a full URL of a piece of content or a page that you want this content to redirect to. If you specify a URL here, this content will not be displayed.

---

# Using OpenJ9 JITServer to offload JIT compilations from Open Liberty applications deployed in Kubernetes

[JITServer Technology](https://www.eclipse.org/openj9/docs/jitserver/) is an [Eclipse OpenJ9 JVM](https://www.eclipse.org/openj9/) feature that allows a user to offload JIT compilations to a remote process which can be containerized and deployed as a service in the cloud. This technique offers numerous advantages as explained in this [IBM Developer article](https://developer.ibm.com/articles/jitserver-optimize-your-java-cloud-native-applications/).

The [Open Liberty Operator](https://openliberty.io/docs/latest/open-liberty-operator.html) can be used to deploy and manage applications running on either [Open Liberty](https://openliberty.io/) or WebSphere Liberty into Kubernetes-based platforms, such as [Red Hat OpenShift](https://www.redhat.com/en/technologies/cloud-computing/openshift). You can also perform Day-2 operations such as gathering traces and dumps using the operator. More details can be found in the [documentation](https://github.com/OpenLiberty/open-liberty-operator/blob/main/doc/user-guide-v1beta2.adoc).

## Learning objectives

In this tutorial we will show you how easy it is to connect an Open Liberty application deployed with the Open Liberty Operator, to a JITServer service deployed with a Helm chart.

## Prerequisites

* [Ubuntu](https://ubuntu.com/) -- Linux OS distribution
* [MicroK8s](https://microk8s.io/) -- Lightweight Kubernetes
* [KVM](https://www.linux-kvm.org/page/Main_Page) -- Kernel Virtual Machine on Linux
* [Podman](https://podman.io/) -- Container engine

For simplicity, the experiments will be performed in a [MicroK8s](https://microk8s.io/) single-node K8s cluster that runs in a KVM virtual machine with Ubuntu 22.04. How to install/configure the VM and MicroK8s is outside the scope of this tutorial. For MicroK8s, the following add-ons were enabled: dns, registry, storage, rbac, ingress, and Prometheus. In Ubuntu, we also installed podman-docker to build the various container images. Note that in MicroK8s all `kubectl` commands need to be prefixed by `microk8s` commands. To simplify development, we have created an alias:

```bash
$ alias kubectl='microk8s kubectl'
```

For the application, we will use the Open Liberty [getting-started app](https://github.com/OpenLiberty/sample-getting-started). The container image can be pulled from the [IBM Cloud Container Registry](https://www.ibm.com/cloud/container-registry) using the following path: **icr.io/appcafe/open-liberty/samples/getting-started**.

## Estimated time

Completing this tutorial should take about 30 minutes.

## Steps

1. [Deploying the JITServer service](#1-deploying-the-jitserver-service)
1. [Install the Open Liberty Operator](#2-install-the-open-liberty-operator)
1. [Deploy the **getting-started** application image](#3-deploy-the-getting-started-application-image)
1. [Clean up](#4-clean-up)

## 1. Deploying the JITServer service

As explained in a [recent tutorial](https://developer.ibm.com/tutorials/using-openj9-jitserver-in-kubernetes/), the JITServer service can be deployed either using yaml files or the Helm chart available in the [openj9-utils](https://github.com/eclipse-openj9/openj9-utils/tree/master/helm-chart/openj9-jitserver-chart) git repository. In this tutorial we will use the second approach and we will loosely follow the steps outlined in this [OpenJ9 blog post](https://blog.openj9.org/2021/03/20/introducing-the-eclipse-openj9-jitserver-helm-chart/).

It is important to note that, for a successful connection between a client JVM and a JITServer instance, the Java version and OpenJ9 release of the two parties must match. Thus, the first step is to determine the version of the OpenJ9 JVM running the Liberty application. However, an easier way to ensure client-server compatibility is to use the image of your Liberty application as a JITServer. This is possible because JITServer is nothing else but an OpenJ9 JVM running in server mode. By adding the “jitserver” argument to the container running the Liberty app (which under the covers runs the OpenJ9 JVM) we are able to morph the Liberty app container into a JITServer container.

Add the repository for the JITServer Helm chart:

```bash
$ microk8s helm3 repo add openj9 https://raw.githubusercontent.com/eclipse/openj9-utils/master/helm-chart/
```

Install the JITServer Helm chart and override the default image repository and default tag with those from the Liberty app image:

```bash
$ microk8s helm3 install myjitserver --set image.repository="icr.io/appcafe/open-liberty/samples/getting-started" --set image.tag="latest" openj9/openj9-jitserver-chart
```

Verify that the JITServer pod is up and running:

```bash
$ kubectl get pods | grep myjitserver
myjitserver-openj9-jitserver-chart-86c8d54d64-mthps   1/1     Running   0          82s
```

Additionally, we can look at the logs of the JITServer instance:

```bash
$ kubectl logs myjitserver-openj9-jitserver-chart-86c8d54d64-mthps

JITServer is ready to accept incoming requests
```

## 2. Install the Open Liberty Operator

### A. Install Custom Resource Definitions (CRDs) for OpenLibertyApplication

This needs to be done only ONCE per cluster:

```bash
$ kubectl apply -f https://raw.githubusercontent.com/OpenLiberty/open-liberty-operator/main/deploy/releases/0.8.2/kubectl/openliberty-app-crd.yaml
```

### B. Set the operator namespace and the namespace to watch

The Open Liberty Operator can be installed to:

1. Watch own namespace
1. Watch another namespace
1. Watch all namespaces in the cluster

Appropriate roles and bindings are required to watch another namespace or watch all namespaces (see the documentation). For simplicity, in this tutorial, the Open Liberty Operator will watch its own namespace, default.

Set the environment variables for the Operator by running the following commands:

```bash
$ OPERATOR_NAMESPACE=default
$ WATCH_NAMESPACE=default
```

### C. Install the operator

```bash
$ curl -L https://raw.githubusercontent.com/OpenLiberty/open-liberty-operator/main/deploy/releases/0.8.2/kubectl/openliberty-app-operator.yaml \
        | sed -e "s/OPEN_LIBERTY_WATCH_NAMESPACE/${WATCH_NAMESPACE}/" \
        | kubectl apply -n ${OPERATOR_NAMESPACE} -f -
```

To check that the Open Liberty Operator has been installed successfully, run the following command to view all the supported API resources that are available through the Open Liberty Operator:

```bash
$ kubectl api-resources --api-group=apps.openliberty.io
```

Look for the following output, which shows the custom resource definitions (CRDs) that can be used by the Open Liberty Operator:

```bash
NAME                      SHORTNAMES         APIGROUP              NAMESPACED   KIND
openlibertyapplications   olapp,olapps       apps.openliberty.io   true         OpenLibertyApplication
openlibertydumps          oldump,oldumps     apps.openliberty.io   true         OpenLibertyDump
openlibertytraces         oltrace,oltraces   apps.openliberty.io   true         OpenLibertyTrace
```

## 3. Deploy the getting-started application image

Create the following YML file which pulls the "getting-started" Open Liberty application image from the IBM Cloud Container Registry. Name the file "liberty.yaml".

```yaml
apiVersion: apps.openliberty.io/v1beta2
kind: OpenLibertyApplication
metadata:
  name: my-liberty-app
spec:
  applicationImage: icr.io/appcafe/open-liberty/samples/getting-started:latest
  env:
  - name: JVM_ARGS
    value: "-XX:+UseJITServer -XX:JITServerAddress=myjitserver-openj9-jitserver-chart -XX:+JITServerLogConnections"
```

Then deploy the getting-started app with the command:

```bash
$ kubectl apply -f liberty.yaml
```

Note the extra options passed to the OpenJ9 JVM through the JVM_ARGS environment variable (this is Open Liberty specific). **-XX:+UseJITServer** instructs OpenJ9 to attempt to connect to a JITServer instance whose address is given by the **-XX:JITServerAddress=** option. This is the end-point of our JITServer service and can be found with:

```bash
$ kubectl get services | grep myjitserver
myjitserver-openj9-jitserver-chart   ClusterIP   10.152.183.217   <none>        38400/TCP   10m
-XX:+JITServerLogConnections tells the client OpenJ9 JVM to print a line whenever it connects to a JITServer instance.
```

Verify that the application started and that it offloads JIT compilations to JITServer by looking at the logs of the application:

```bash
$ kubectl get pods | grep my-liberty-app
my-liberty-app-bb96dbd78-tq68r                        1/1     Running   0          3m

$ kubectl logs my-liberty-app-bb96dbd78-tq68r | grep JITServer
#JITServer: t=    10 Connected to a server (serverUID=11873153734538590431)
```

## 4. Clean up

Uninstall the getting-started application by using the kubectl delete command on the liberty.yaml file from Step 3:

```bash
$ kubectl delete -f liberty.yaml
```

Uninstall the Open Liberty Operator and the corresponding CRDs by running commands from Step 2.C and 2.A after replacing kubectl apply with kubectl delete:

```bash
$ curl -L https://raw.githubusercontent.com/OpenLiberty/open-liberty-operator/main/deploy/releases/0.8.2/kubectl/openliberty-app-operator.yaml \
        | sed -e "s/OPEN_LIBERTY_WATCH_NAMESPACE/${WATCH_NAMESPACE}/" \
        | kubectl delete -n ${OPERATOR_NAMESPACE} -f -
$ kubectl delete -f https://raw.githubusercontent.com/OpenLiberty/open-liberty-operator/main/deploy/releases/0.8.2/kubectl/openliberty-app-crd.yaml
```

Remove the JITServer service and deployment:

```bash
$ microk8s helm3 delete myjitserver
```

## Summary

In this tutorial we showed how to connect an Open Liberty application deployed with the Open Liberty Operator to a JITServer service deployed with a Helm chart.

In order to ensure compatibility between the JVM running the Liberty app and JITServer, we took a shortcut and picked the JITServer container image to be identical to the Liberty app container image. This approach does not increase the runtime footprint or CPU consumption of JITServer, because it is going to use only the JDK from the Liberty app image. Moreover, there is no increase in terms of disk space because the same container image is reused for two different purposes. However, this technique forces you to have a separate JITServer deployment for each Liberty app. If your goal is to maximize container density in the cloud (including JITServer instances), then it may be better to use a separate JITServer image based on OpenJ9 (as shown in this tutorial) and connect multiple Open Liberty applications to it.
