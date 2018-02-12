---
title: "Kubernetes CNI"
date: 2018-01-29T06:58:17Z
draft: true
weight: 1
---

I have spent the last year helping Google's Cloud Platform customer's troubleshoot
their Kubernetes clusters. Despite all of this experience, k8s networking still 
felt to me like it is being held together by black magic and duct tape so this
last weekend, I decided it was time to get serious about figuring out how it
works. Enter CNI.

### What is CNI?

CNI stands for "container network interface." In practice, CNI is exactly what it
sounds like: a method by which container's gain a network interface. How that
interface is created on the other hand is not so straight forward.

The CNI is a [CNCF](https://www.cncf.io/) 
[project](https://github.com/containernetworking/cni) to make container networking
easier in large clusters by providing a standardized API for plugins to
manage a container's network interface. What does all that mean? It means that a
single application can automate IP address allocation, subnetting, and routing.
It goes without saying that for a resource scheduler such as Kubernetes, having
full control of the network stack is imperative.

### Terms to understand

Before I go over what CNI is, you will need to understand a few basic terms.

First, the term container in the context of CNI applies to a single network 
namespace. This namespace can (and often does in the case of k8s pods) provide
the networking for multiple Docker containers.

Second, the term network refers to a group of entities that are uniquely addressable
and routable. These entities can be a single container as specified above, a 
machine, or even networking equipment such as routers. Container's (again, as 
defined above) can be added to and removed from a network at will using CNI.

### How does CNI work?

There are three major components to a system that has successfully implemented CNI:
the network manager (for lack of a better word), the plugin, and the container
runtime. The container runtime is charged with creating and maintaining the
container (in this case, k8s pod abstraction), the plugin is charged with
configuring the container's interface, and the network manager is charged with
maintaining the routing tables

Let's start by setting out a solid environment to make visualizing the components
and interactions easy. In the case of a k8s deployment running flannel, flannel
acts as the network manager, the kubelet provides the container runtime, and 
flannel provides the plugin binary.

    Note that the flannel plugin is actually maintained as one of the core CNI
    plugins so if you are installing k8s using kubeadm, you'll get the plugin
    with the kubernetes-cni package.
    
