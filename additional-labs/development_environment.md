# OpenShift development environment

This page shows different ways to test self-developed containers or OpenShift templates etc. without having access to a full, production OpenShift platform like e.g. APPUiO.

## CodeReady Containers

[CodeReady Containers](https://crc.dev/crc/) makes it possible to run a minimal OpenShift 4 cluster on the local machine.

## Minishift

[Minishift](https://docs.okd.io/3.11/minishift/index.html) allows you to run a local OpenShift installation on your own notebook in a VM using KVM, Hyper-V or VirtualBox, __but only allows you to use OpenShift 3, not OpenShift 4__.
Minishift is originally a fork of Minikube and uses [OKD](https://www.okd.io/), the upstream project of OpenShift Container Platform.
For the use of OCP 3, the Red Hat CDK must be used.

## CDK

The [Red Hat Container Development Kit](https://developers.redhat.com/products/cdk/overview) offers the "enterprise" variant of Minishift, so to speak.
Instead of OKD, OCP is used, which is why a subscription is required, e.g. via the free [Red Hat Developer Program](https://developers.redhat.com/).

---

__End__


[← back to the overview](../README.md)

