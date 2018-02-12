---
title: "Kubelet"
date: 2018-01-31T05:23:41Z
draft: true
---

{{% gist thefirstofthe300 0464494e31525f0ae14d179acf70aedd %}}

Currently, the kubelet starts up the gRPC CRI server:
https://github.com/kubernetes/kubernetes/blob/v1.9.2/pkg/kubelet/kubelet.go#L628

The dockershim acts as one of two defined gRPC servers.

The kubelet will then watch three different places for new pods: the manifest directory,
an HTTP endpoint, and it's internal (and lacking in features) HTTP server, as well as
the API server. The kubelet receives updates from the API server by executing
a watch (hanging GET) on the API server and filtering for only pods scheduled to
the node on which the kubelet resides:
https://github.com/kubernetes/kubernetes/blob/v1.9.2/pkg/kubelet/kubelet.go#L314
https://github.com/kubernetes/kubernetes/blob/v1.9.2/pkg/kubelet/config/apiserver.go#L33
https://github.com/kubernetes/client-go/blob/21ab0aa61a13eb1ea583b24c69943f9fea5929bd/tools/cache/listwatch.go#L64
https://github.com/kubernetes/client-go/blob/8a8517e82fc13125243513ecac9aaf98789ced90/tools/cache/reflector.go#L92

Of note, the API server IP is passed to the kubelet in the kubeconfig:

It is in charge of spinning pods up and down based on the passed pod configuration:
https://github.com/kubernetes/kubernetes/blob/v1.9.2/pkg/kubelet/dockershim/docker_service.go#L265

The kubelet is in charge of generating a complete pod config based on the pod yaml as well as any
relevant kubelet options that were set.

The PodSandbox is generally the "pause" container, is created before all other
containers, and is the sandbox ID corresponds to the network namespace:
https://github.com/kubernetes/kubernetes/blob/v1.9.2/pkg/kubelet/dockershim/docker_sandbox.go#L78

Along the way, the resolv.conf file for the sandbox container is rewritten:
https://github.com/kubernetes/kubernetes/blob/v1.9.2/pkg/kubelet/dockershim/docker_sandbox.go#L141

The nameservers, search, and option fields are all written based on the passed
pod config.

The CRI shim then makes a call to the CNI plugin's SetUpPod() function:
https://github.com/kubernetes/kubernetes/blob/v1.9.2/pkg/kubelet/network/plugins.go#L68

The SetUpPod function is a call to the CNI network plugin:
https://github.com/kubernetes/kubernetes/blob/v1.9.2/pkg/kubelet/network/cni/cni.go#L208

Basically, it creates a proper CNI config using the hardcoded file paths and
then invokes CNI to add the pod to the network:
https://github.com/kubernetes/kubernetes/blob/v1.9.2/pkg/kubelet/network/cni/cni.go#L225

CNI execution occurs using libcni's API:
https://github.com/containernetworking/cni/blob/75a5cf1bf039bd3cb5c22751df93d616c2e01dd2/pkg/invoke/exec.go#

Once pod creation has succeeded, containers can be added to and removed from the pod at will.



Due to the nature of CNI, pod deletion should occur in almost a perfect inverse manner.