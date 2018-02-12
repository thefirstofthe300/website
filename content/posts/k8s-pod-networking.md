---
title: "Kubernetes Pod Networking"
date: 2018-01-30T07:44:41Z
draft: true
weight: 1
---

If you have done any work with Kubernetes, you probably understand that the k8s
networking stack is not a simple one. There are a lot of moving parts and 
understanding all of those parts can be lot to take in.

I've personally been working with Kubernetes now for a little over a year and 
finally decided it was time I sit down and learn exactly what mechanisms k8s 
uses to make a request magically traverse the network from a load balancer external
to the cluster to a pod and back.

At the heart of Kubernetes lies the pod so understanding how a pod's networking
is configured is probably a good place to start.

I figure this is going to be a multipart series of posts dealing with a different
aspect of the network so here goes.

### Pod networking

Pod's are unique in Kubernetes in that they consist of multiple Docker containers
yet, in a default pod configuration, they are all a member of the same network
namespace. By default, each Docker container has it's own namespace so this 
deviation is fairly significant. In addition, each pod is assigned an IP address
that is routable from the host and all other pods in the cluster. What exactly
does these two things mean for the containers in a pod?

First, it means that all of the containers in a pod share the same localhost. For example,
an Nginx reverse proxy container and an application web server in the same pod
can communicate with each other directly using 127.0.0.1 or localhost. This also
means that the Docker container's in a single pod have to coordinate ports, i.e the
aforementioned Nginx proxy and the application server can't both serve traffic 
on port 80 but this behavior is no different than coordinating ports on a physical
machine.

Second, having a routable IP address means that networking stays simpler as the
