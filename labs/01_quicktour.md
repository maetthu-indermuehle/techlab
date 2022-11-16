# Lab 1: Quicktour through OpenShift

In this lab, we will introduce the basic concepts of OpenShift. Furthermore, we will show how to log into the Web Console and briefly introduce the individual areas.

The terms and resources listed here are an excerpt [from this Red Hat blog post](https://developers.redhat.com/blog/2018/02/22/container-terminology-practical-introduction/), which also provides further information on container-related terms.


## Basic concepts

OpenShift is based on modern concepts such as CRI-O or Kubernetes and thus offers a platform with which software can be built, deployed and operated in containers.


### Container Engine

[cri-o](https://cri-o.io/) is a lightweight container engine that converts container images into a running process, i.e. a container. This is done using the [Container Runtime Specification](https://github.com/opencontainers/runtime-spec) of the [Open Container Initiative](https://www.opencontainers.org/), which has also defined the [Image Specification](https://github.com/opencontainers/image-spec). All OCI-compliant images can thus be executed with OCI-compliant engines.


### Kubernetes

[Kubernetes](http://kubernetes.io/) is a container orchestration tool that makes managing containers much easier. The orchestrator dynamically schedules the container workload within a cluster.


### Container and Container Images

The basic elements of OpenShift applications are containers. A container is an isolated process on a Linux system with limited resources that can only interact with defined processes.

Containers are based on container images. A container is started by the container engine unpacking the files and metadata of the container image and passing them to the Linux kernel.

Container images are built, for example, using Dockerfiles (textual description of how the container image is built step by step). Basically, container images are hierarchically applied filesystem snapshots.

Example of Tomcat:

- Base Image (z.B. [UBI](https://www.redhat.com/en/blog/introducing-red-hat-universal-base-image))
- \+ Java
- \+ Tomcat
- \+ App

Built container images are versioned in an image registry (analogous to a repository for e.g. RPM packages) and can be obtained from there to deploy them on a container platform.
Container images can also be built on OpenShift itself, from where they are pushed to the OpenShift internal registry and pulled again for deployment.


## OpenShift-Konzepte
### Projects

In OpenShift, resources (containers and container images, pods, services, routes, configuration, quotas and limits, etc.) are structured in projects. From a technical perspective, a project corresponds to a Kubernetes namespace and extends it with certain concepts.

Within a project, authorized users can manage and organize their resources themselves.

The resources within a project are connected via a transparent [SDN](https://de.wikipedia.org/wiki/Software-defined_networking) or overlay network. Thus, the individual components of a project can be deployed to different nodes in a multi-node setup. They are visible and accessible to each other via the SDN.


### Pods

OpenShift adopts the concept of pods from Kubernetes.

A pod is one or more containers deployed together on the same host. A pod is the smallest manageable unit in OpenShift.

A pod is available within an OpenShift project via the corresponding service, among others.


### Services

A service represents an internal load balancer to the pods behind it (replicas of the same type). The service serves as a proxy to the pods and forwards requests to them. Thus, pods can be added and removed from a service arbitrarily while the service remains available.

A service is assigned an IP and a port within a project.


### Routes

With a route, one defines in OpenShift how a service can be reached from outside OpenShift by external clients.

These routes are entered in the integrated routing layer and then allow the platform to forward the requests to the corresponding service via a hostname mapping.

If more than one pod is deployed for a service, the routing layer distributes the requests to the deployed pods.

The following protocols are currently supported:

- HTTP
- HTTPS ([SNI](https://en.wikipedia.org/wiki/Server_Name_Indication))
- WebSockets
- TLS-encrypted protocols with [SNI](https://en.wikipedia.org/wiki/Server_Name_Indication)


### Templates

A template textually describes a list of resources that can be run on OpenShift and created in OpenShift accordingly.

This gives the possibility to describe whole infrastructures:

- Java Application Service (3 replicas, rolling upgrade)
- Database Service
- Available on the Internet via route java.app.appuio-beta.ch
- ...

---

__Ende Lab 1__

<p width="100px" align="right"><a href="02_cli.md">OpenShift CLI installation →</a></p>

[← back to the overview](../README.md)
