---
title: "Deploying HA Cluster-API-Provider-vSphere Kubernetes Clusters with CAPI version v1alpha2"
date: 2020-01-26T23:38:58-06:00
tags:
  - "CAPV"
  - "Cluster-API"
  - "Kubernetes" 
draft: false
thumbnail: "img/capi.jpeg"
toc: false
---

In this post I'll cover standing up a v1alpha2 cluster-api-vsphere based HA cluster on vSphere 6.7U3. There are a lot of posts out there that talk about running Kubernetes on vSphere leveraging CAPV, but I'm unable to find anything that represents a more robust HA control-plane deployment. Like this:

![](https://i.imgur.com/4g8e4jN.png)

A NOTE: This guide is for v1alpha2 clusters. Currently work is being done in the cluster-api-provider-vsphere repository to enable v1alpha3 features. The master branch of the repository can be broken at times. I recommend [using the 0.5.4 release](https://github.com/kubernetes-sigs/cluster-api-provider-vsphere/releases) which maps to v1alpha2 for stability's sake.

## Cluster API Provider vSphere basics

For those unfamiliar with cluster-api-vsphere, I recommend checking out the getting started guide from the [cluster-api-vsphere repository](https://github.com/kubernetes-sigs/cluster-api-provider-vsphere/blob/master/docs/getting_started.md) For a higher level overview of CAPV review [this blog by Chris Milstead](https://blogs.vmware.com/cloudnative/2019/12/12/how-cluster-api-promotes-self-service-infrastructure/) and [this blog by Eric Shanks](https://theithollow.com/2019/11/04/clusterapi-demystified/).

As mentioned in Chris' blog, an implementation of an initial Kubernetes control-plane node for vSphere requires `KubeadmConfig`, `Machine`, and `VSphereMachine` objects to be created in a cluster-api management cluster. A framework for creating these objects is provided by leveraging the [quickstart's documentation](https://github.com/kubernetes-sigs/cluster-api-provider-vsphere/blob/master/docs/getting_started.md). Nearly all posts stop here and move on to worker node `MachineDeployment` configuration. In this post, we'll cover what else is needed to deploy an HA control-plane.

## What's needed to complete the HA setup?

To configure an HA control-plane, we'll add two additional control plane nodes and a load balancer. Here is a list of the changes needed:

1. Two additional control plane nodes. 
2. A load balancer for the Kubernetes API server on each of the three control-plane nodes.
4. Static IP's for your control plane nodes.
5. KubeadmConfigs for the joining control plane nodes.

## Load balancing

Kubernetes doesn't care about the load balancer in front of the api server, as long as it forwards TCP traffic to port 6443 (port can be altered to taste). In my lab, I'm using Nginx to provide load balancing from a static IP address to static IP based control-plane nodes. The setup looks like this:

![](https://i.imgur.com/OF5VyZh.jpg)

My nginx.conf looks like this:
```
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

stream {
        upstream kube_api  {
          server 192.168.10.31:6443;
          server 192.168.10.32:6443;
          server 192.168.10.33:6443;
        }
        server {
          listen        6443;
          proxy_pass    kube_api;
        }
}
```

To enable use of this load balancer configuration edits are needed to the controlplane.yaml as created by following the [instructions here](https://github.com/kubernetes-sigs/cluster-api-provider-vsphere/blob/master/docs/getting_started.md#managing-workload-clusters-using-the-management-cluster) Edit the `KubeadmConfig` of the controlplane.yaml:

1. Add apiServerCertSANS to the clusterConfiguration spec
2. Add a controlPlaneEndpoint to the clusterConfiguration spec 

Both should be configured to leverage the IP and port of the load balancer configured to support the control-plane. An example of this full configuration from my lab is below:
```
apiVersion: bootstrap.cluster.x-k8s.io/v1alpha2
kind: KubeadmConfig
metadata:
  name: capv-5-controlplane-0
  namespace: default
spec:
  clusterConfiguration:
    apiServer:
      extraArgs:
        cloud-provider: external
    apiServerCertSANS:
    - 192.168.10.18
    controllerManager:
      extraArgs:
        cloud-provider: external
    controlPlaneEndpoint: 192.168.10.18:6443
    imageRepository: k8s.gcr.io
  initConfiguration:
    nodeRegistration:
      criSocket: /var/run/containerd/containerd.sock
      kubeletExtraArgs:
        cloud-provider: external
      name: '{{ ds.meta_data.hostname }}'
  preKubeadmCommands:
  - hostname "{{ ds.meta_data.hostname }}"
  - echo "::1         ipv6-localhost ipv6-loopback" >/etc/hosts
  - echo "127.0.0.1   localhost {{ ds.meta_data.hostname }}" >>/etc/hosts
  - echo "{{ ds.meta_data.hostname }}" >/etc/hostname
  users:
  - name: capv
    sshAuthorizedKeys:
    - "The public side of an SSH key pair."
    sudo: ALL=(ALL) NOPASSWD:ALL
```
## Control-Plane Node Static IP Assignment

I'm using Ubuntu images in my lab as I found some open GitHub issues that referenced static IP configuration problems in CentOS related to the upstream [image-builder](https://github.com/kubernetes-sigs/image-builder) tooling for kubernetes. If you're new to the Kubernetes space on vSphere and can use Ubuntu my recommendation would be do so for now.

To configure static IP's for control plane nodes edit the related `VSphereMachine` network spec setting dhcp4 to false, adding gateway4, ipAddrs, nameservers, and searchDomains. Here's how this section of my controlplane.yaml ends up looking:

```
---
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha2
kind: VSphereMachine
metadata:
  labels:
    cluster.x-k8s.io/cluster-name: management-cluster
    cluster.x-k8s.io/control-plane: "true"
  name: capv-5-cluster-controlplane-0
  namespace: default
spec:
  datacenter: lab
  diskGiB: 50
  memoryMiB: 4096
  network:
    devices:
    - dhcp4: false
      dhcp6: false
      gateway4: 192.168.10.1
      ipAddrs:
      - 192.168.10.31/24
      nameservers:
      - 192.168.10.50
      - 192.168.10.51
      networkName: management
      searchDomains: 
      - timcarr.net
  numCPUs: 4
  template: ubuntu-1804-kube-v1.16.3
```
Your controlplane.yaml file should include configurations for `Machine`, `VSphereMachine`, and the updated `KubeadmConfig`. These three configurations define a how the single controlplane-0 VM are configured in vSphere (`Machine` and `VSphereMachine`), and bootstrapped (`KubeadmConfig`) in Kubernetes leveraging our external load balancer. Applying this configuration to your CAPV enabled management cluster should provision that single node.

## Adding Additional Control Plane Nodes

For each of the additional two control-plane nodes, `Machine`,`VSphereMachine`, and `KubeadmConfig` specifying a `joinConfiguration` objects are needed.

### `Machine` settings
Each control-plane `machine` configuration needs to contain a label `cluster.x-k8s.io/control-plane: "true"` that informs cluster-api that the machine specified is a part of the referenced cluster's control-plane. You'll also note that the `Machine` object references bootstrap (`KubeadmConfig`) and infrastructure (`VSphereMachine`) objects. Here's an example from my controlplane.yaml file including the needed label:

```
---
apiVersion: cluster.x-k8s.io/v1alpha2
kind: Machine
metadata:
  labels:
    cluster.x-k8s.io/cluster-name: capv-5
    cluster.x-k8s.io/control-plane: "true"
  name: capv-5-controlplane-1
  namespace: default
spec:
  bootstrap:
    configRef:
      apiVersion: bootstrap.cluster.x-k8s.io/v1alpha2
      kind: KubeadmConfig
      name: capv-5-controlplane-1
      namespace: default
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1alpha2
    kind: VSphereMachine
    name: capv-5-controlplane-1
    namespace: default
  version: 1.16.3
```

### `VSphereMachine` settings 
Each`VSphereMachine` configuration requires same `cluster.x-k8s.io/control-plane: "true"` as well as changes to specify a static IP. Here's an example from my controlplane.yaml file:
```
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha2
kind: VSphereMachine
metadata:
  labels:
    cluster.x-k8s.io/cluster-name: capv-5
    cluster.x-k8s.io/control-plane: "true"
  name: capv-5-controlplane-1
  namespace: default
spec:
  datacenter: lab
  diskGiB: 50
  memoryMiB: 2048
  network:
    devices:
    - dhcp4: false
      dhcp6: false
      gateway4: 192.168.10.1
      ipAddrs:
      - 192.168.10.32/24
      nameservers:
      - 192.168.10.50
      - 192.168.10.51
      networkName: management
      searchDomains: 
      - timcarr.net
  numCPUs: 2
  template: ubuntu-1804-kube-v1.16.3
```
### `KubeadmConfig` settings
The `KubeadmConfig` setting here differs from the initial configuration for control-plane-0 as it just tells subsequent nodes that they should join the cluster using the included `joinConfiguration`. Here is an example of my configuration.
```
---
apiVersion: bootstrap.cluster.x-k8s.io/v1alpha2
kind: KubeadmConfig
metadata:
  name: capv-5-controlplane-1
  namespace: default
spec:
  joinConfiguration:
    nodeRegistration:
      criSocket: /var/run/containerd/containerd.sock
      kubeletExtraArgs:
        cloud-provider: external
      name: '{{ ds.meta_data.hostname }}'
  preKubeadmCommands:
  - hostname "{{ ds.meta_data.hostname }}"
  - echo "::1         ipv6-localhost ipv6-loopback" >/etc/hosts
  - echo "127.0.0.1   localhost {{ ds.meta_data.hostname }}" >>/etc/hosts
  - echo "{{ ds.meta_data.hostname }}" >/etc/hostname
  users:
  - name: capv
    sshAuthorizedKeys:
    - "The public side of an SSH key pair."
    sudo: ALL=(ALL) NOPASSWD:ALL
```

## Wrapping it up.

At this point you should be able to copy the exact same settings above and modify the names in the configuration files and IP addresses to provide a third node. 

Here's a final version of my controlplane.yaml with all of these objects defined omitting my public SSH key:
```
apiVersion: bootstrap.cluster.x-k8s.io/v1alpha2
kind: KubeadmConfig
metadata:
  name: capv-5-controlplane-0
  namespace: default
spec:
  clusterConfiguration:
    apiServer:
      extraArgs:
        cloud-provider: external
    apiServerCertSANS:
    - 192.168.10.18
    controllerManager:
      extraArgs:
        cloud-provider: external
    controlPlaneEndpoint: 192.168.10.18:6443
    imageRepository: k8s.gcr.io
  initConfiguration:
    nodeRegistration:
      criSocket: /var/run/containerd/containerd.sock
      kubeletExtraArgs:
        cloud-provider: external
      name: '{{ ds.meta_data.hostname }}'
  preKubeadmCommands:
  - hostname "{{ ds.meta_data.hostname }}"
  - echo "::1         ipv6-localhost ipv6-loopback" >/etc/hosts
  - echo "127.0.0.1   localhost {{ ds.meta_data.hostname }}" >>/etc/hosts
  - echo "{{ ds.meta_data.hostname }}" >/etc/hostname
  users:
  - name: capv
    sshAuthorizedKeys:
    - "The public side of an SSH key pair."
    sudo: ALL=(ALL) NOPASSWD:ALL
---
apiVersion: cluster.x-k8s.io/v1alpha2
kind: Machine
metadata:
  labels:
    cluster.x-k8s.io/cluster-name: capv-5
    cluster.x-k8s.io/control-plane: "true"
  name: capv-5-controlplane-0
  namespace: default
spec:
  bootstrap:
    configRef:
      apiVersion: bootstrap.cluster.x-k8s.io/v1alpha2
      kind: KubeadmConfig
      name: capv-5-controlplane-0
      namespace: default
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1alpha2
    kind: VSphereMachine
    name: capv-5-controlplane-0
    namespace: default
  version: 1.16.3
---
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha2
kind: VSphereMachine
metadata:
  labels:
    cluster.x-k8s.io/cluster-name: capv-5
    cluster.x-k8s.io/control-plane: "true"
  name: capv-5-controlplane-0
  namespace: default
spec:
  datacenter: lab
  diskGiB: 50
  memoryMiB: 2048
  network:
    devices:
    - dhcp4: false
      dhcp6: false
      gateway4: 192.168.10.1
      ipAddrs:
      - 192.168.10.31/24
      nameservers:
      - 192.168.10.50
      - 192.168.10.51
      networkName: management
      searchDomains: 
      - timcarr.net
  numCPUs: 2
  template: ubuntu-1804-kube-v1.16.3
---
apiVersion: bootstrap.cluster.x-k8s.io/v1alpha2
kind: KubeadmConfig
metadata:
  name: capv-5-controlplane-1
  namespace: default
spec:
  joinConfiguration:
    nodeRegistration:
      criSocket: /var/run/containerd/containerd.sock
      kubeletExtraArgs:
        cloud-provider: external
      name: '{{ ds.meta_data.hostname }}'
  preKubeadmCommands:
  - hostname "{{ ds.meta_data.hostname }}"
  - echo "::1         ipv6-localhost ipv6-loopback" >/etc/hosts
  - echo "127.0.0.1   localhost {{ ds.meta_data.hostname }}" >>/etc/hosts
  - echo "{{ ds.meta_data.hostname }}" >/etc/hostname
  users:
  - name: capv
    sshAuthorizedKeys:
    - "The public side of an SSH key pair."
    sudo: ALL=(ALL) NOPASSWD:ALL
---
apiVersion: cluster.x-k8s.io/v1alpha2
kind: Machine
metadata:
  labels:
    cluster.x-k8s.io/cluster-name: capv-5
    cluster.x-k8s.io/control-plane: "true"
  name: capv-5-controlplane-1
  namespace: default
spec:
  bootstrap:
    configRef:
      apiVersion: bootstrap.cluster.x-k8s.io/v1alpha2
      kind: KubeadmConfig
      name: capv-5-controlplane-1
      namespace: default
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1alpha2
    kind: VSphereMachine
    name: capv-5-controlplane-1
    namespace: default
  version: 1.16.3
---
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha2
kind: VSphereMachine
metadata:
  labels:
    cluster.x-k8s.io/cluster-name: capv-5
    cluster.x-k8s.io/control-plane: "true"
  name: capv-5-controlplane-1
  namespace: default
spec:
  datacenter: lab
  diskGiB: 50
  memoryMiB: 2048
  network:
    devices:
    - dhcp4: false
      dhcp6: false
      gateway4: 192.168.10.1
      ipAddrs:
      - 192.168.10.32/24
      nameservers:
      - 192.168.10.50
      - 192.168.10.51
      networkName: management
      searchDomains: 
      - timcarr.net
  numCPUs: 2
  template: ubuntu-1804-kube-v1.16.3
---
apiVersion: bootstrap.cluster.x-k8s.io/v1alpha2
kind: KubeadmConfig
metadata:
  name: capv-5-controlplane-2
  namespace: default
spec:
  joinConfiguration:
    nodeRegistration:
      criSocket: /var/run/containerd/containerd.sock
      kubeletExtraArgs:
        cloud-provider: external
      name: '{{ ds.meta_data.hostname }}'
  preKubeadmCommands:
  - hostname "{{ ds.meta_data.hostname }}"
  - echo "::1         ipv6-localhost ipv6-loopback" >/etc/hosts
  - echo "127.0.0.1   localhost {{ ds.meta_data.hostname }}" >>/etc/hosts
  - echo "{{ ds.meta_data.hostname }}" >/etc/hostname
  users:
  - name: capv
    sshAuthorizedKeys:
    - "The public side of an SSH key pair."
    sudo: ALL=(ALL) NOPASSWD:ALL
---
apiVersion: cluster.x-k8s.io/v1alpha2
kind: Machine
metadata:
  labels:
    cluster.x-k8s.io/cluster-name: capv-5
    cluster.x-k8s.io/control-plane: "true"
  name: capv-5-controlplane-2
  namespace: default
spec:
  bootstrap:
    configRef:
      apiVersion: bootstrap.cluster.x-k8s.io/v1alpha2
      kind: KubeadmConfig
      name: capv-5-controlplane-2
      namespace: default
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1alpha2
    kind: VSphereMachine
    name: capv-5-controlplane-2
    namespace: default
  version: 1.16.3
---
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha2
kind: VSphereMachine
metadata:
  labels:
    cluster.x-k8s.io/cluster-name: capv-5
    cluster.x-k8s.io/control-plane: "true"
  name: capv-5-controlplane-2
  namespace: default
spec:
  datacenter: lab
  diskGiB: 50
  memoryMiB: 2048
  network:
    devices:
    - dhcp4: false
      dhcp6: false
      gateway4: 192.168.10.1
      ipAddrs:
      - 192.168.10.33/24
      nameservers:
      - 192.168.10.50
      - 192.168.10.51
      networkName: management
      searchDomains: 
      - timcarr.net
  numCPUs: 2
  template: ubuntu-1804-kube-v1.16.3
  ```
In the end your vSphere environment should look like this:
![](https://i.imgur.com/rzjjFjs.png)

One final note. I've setup my lab to ensure that my management VM has a static IP - you should know how to do that now! Hope this helps everyone looking to [get started](https://github.com/kubernetes-sigs/cluster-api-provider-vsphere/blob/master/docs/getting_started.md) with [Cluster-API-Provider-vSphere](https://github.com/kubernetes-sigs/cluster-api-provider-vsphere).
