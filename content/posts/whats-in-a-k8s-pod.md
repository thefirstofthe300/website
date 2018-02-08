---
title: "What's in a Kubernetes Pod?"
date: 2018-02-08T07:15:00Z
draft: false
---

Kubernetes is a great piece of technology and at its core is a resource with
some very interesting characteristics: the pod. In my current job, I have been
supporting Kubernetes for over a year, so in the last week, I finally decided to
do a deep dive into what makes a pod special and the underlying mechanisms the
pod uses to function.

### So what is a pod?

Let's start off with the basics of what makes a Kubernetes pod unique. Each pod
is a container or group of containers that "live and die" together, share the
same localhost, and are able to communicate with each other via standard IPC 
interfaces such as POSIX semaphores or shared memory. In addition, the pod
abstraction allows for things such as data volume mounts to be decoupled from
the container lifecycle and instead tied to the lifecycle of the pod.

Obviously, these characteristics make pods very flexible and well-suited to 
containerize all types of applications; but, given that Kubernetes
relies heavily on Docker, I always wondered how Kubernetes constructed
a pod. I finally decided to dig into the k8s codebase to answer my question, so
here's my braindump on what I found in a step-by-step description of how a pod is
created in a Kubernetes 1.9.2 cluster.

The pod starts out with a configuration change on the master that declares a new
pod should be created on a node. The kubelet's on each node watch the API server
for changes in the pod configurations using a "hanging GET". When a kubelet sees
that a new pod needs to be constructed on it's node, it kicks off a new pod
creation.

### The container runtime interface

Kubernetes uses an abstraction called the Container Runtime Interface (CRI) to 
abstract away the implementation details of the containerization software, for
example, Docker, rkt, or lxd. When complete, it will allow the kubelet to support
multiple container runtimes without the need to recompile the kubelet binary. 
The CRI implementation is currently still incomplete so the only supported 
container runtime is Docker and the CRI server is still bundled in the kubelet.

### The pod sandbox

Before creating any of the containers defined in the pod configuration, the
kubelet creates the pod sandbox. The sandbox is an infrastructure pod, normally
named pause, used to hold the network, IPC, and, in the most recent versions of
Kubernetes, PID namespaces. It also serves as the "zombie janitor" as described
in detail over on [Ian Lewis's blog](https://www.ianlewis.org/en/almighty-pause-container).

To create this sandbox, the kubelet makes an RPC call to the CRI server with
details of the pod configuration. The CRI abstraction then makes the runtime specific
calls to pull the image for the infrastructure container, creates the container,
gets the network namespace ID, rewrites the infra pod's resolv.conf file with the DNS
information provided by the master, and invokes the CNI plugin to allocate the
pod's IP and add the container to the software-defined pod network.

At this point, you should have a healthy pod sandbox to which you can add your
user-defined containers; however, as is, the pod suffers from a fairly major
limitation: the IP address allocated to the pod is not routable outside
of the node due to the way CNI works. An SDN such as flannel is required to
accomplish routing. I plan to make a future post discussing the CNI implementation
and how Kubernetes uses it in combination with SDN to make pods routable.