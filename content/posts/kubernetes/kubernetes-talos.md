+++
author = "Lewis Green"
title = "Kubernetes using Talos Linux"
date = "2023-05-07"
description = "How to setup a Kubernetes cluster using Talos Linux, FluxCD"
tags = [
    "kubernetes",
    "talos",
    "fluxcd",
    "cilium"
]
+++

# Overview

How to use purely GitOps to build a Kubernetes cluster, with Rook Ceph as the storage, FluxCD for automation, Cilium for the CNI, and Talos Linux as the OS.

![Diagram](/images/kubernetes-talos-diagram.svg)


# Cluster Build

We will have three nodes, which will all be control plane and worker nodes in this setup.

|**Node**|**IP**|k 
|--|--|
| geb | 172.16.1.11 |
| shu | 172.16.1.12 |
| kek | 172.16.1.13 |

There is a tool called talhelper at https://github.com/budimanjojo/talhelper which is good to use as it will allow DRY configuration files 
and auto generate the configuration files per node.


## talhelper sample files

**talconfig.yaml**
```yaml
clusterName: ${clusterName}

talosVersion: v1.4.1
kubernetesVersion: v1.27.1
endpoint: https://${clusterName}.stanleygrange.uk:6443

domain: cluster.local

allowSchedulingOnMasters: true

additionalMachineCertSans:
  - ${clusterEndpointIP}
  - ${clusterName}.stanleygrange.uk
additionalApiServerCertSans:
  - ${clusterEndpointIP}
  - ${clusterName}.stanleygrange.uk

clusterPodNets:
  - 10.244.0.0/16
clusterSvcNets:
  - 10.96.0.0/12

cniConfig:
  name: none

nodes:
  - hostname: geb.stanleygrange.uk
    ipAddress: 172.16.1.11
    installDisk: /dev/sda
    controlPlane: true
    disableSearchDomain: true
    kernelModules:
      - name: br_netfilter
        parameters:
          - nf_conntrack_max=131072
    networkInterfaces:
      - interface: eth0
        dhcp: true
  - hostname: shu.stanleygrange.uk
    ipAddress: 172.16.1.12
    installDisk: /dev/sda
    controlPlane: true
    disableSearchDomain: true
    kernelModules:
      - name: br_netfilter
        parameters:
          - nf_conntrack_max=131072
    networkInterfaces:
      - interface: eth0
        dhcp: true
  - hostname: kek.stanleygrange.uk
    ipAddress: 172.16.1.13
    installDisk: /dev/sda
    controlPlane: true
    disableSearchDomain: true
    kernelModules:
      - name: br_netfilter
        parameters:
          - nf_conntrack_max=131072
    networkInterfaces:
      - interface: eth0
        dhcp: true
controlPlane:
  patches:
    - |-
      cluster:
        allowSchedulingOnMasters: true

    - |-
      machine:
        files:
          - op: create
            path: /etc/cri/conf.d/20-customization.part
            content: |
              [plugins]
                [plugins."io.containerd.grpc.v1.cri"]
                  enable_unprivileged_ports = true
                  enable_unprivileged_icmp = true
        kubelet:
          extraArgs:
            feature-gates: CronJobTimeZone=true,GracefulNodeShutdown=true
#            rotate-server-certificates: "true"
#        network:
#          extraHostEntries:
#            - ip: ${clusterEndpointIP}
#              aliases:
#                - ${clusterName}.stanleygrange.uk
#        sysctls:
#          fs.inotify.max_user_watches: "1048576"
#          fs.inotify.max_user_instances: "8192"
        time:
          disabled: false
          servers:
            - 176.58.109.199
```

Once we have this config file defined then we can generate the Talos configuration files by running the command

```bash
talhelper genconfig
```

This will make the files in a new folder called clusterconfig. We can now apply the configuration files to the nodes.

```bash
cd clusterconfig
talosctl apply-config -i -n 172.16.1.11 -f cluster-0-geb.stanleygrange.uk.yaml
talosctl apply-config -i -n 172.16.1.12 -f cluster-0-shu.stanleygrange.uk.yaml
talosctl apply-config -i -n 172.16.1.13 -f cluster-0-kek.stanleygrange.uk.yaml
```

Once the machines are configured we can bootstrap a node, which does them all.

```bash
talosctl bootstrap -n 172.16.1.11
```

Then we can get the kubeconfig for the cluster

```bash
talosctl kubeconfig -n 172.16.1.11
```

If you run ```kubectl get nodes``` they will show as NotReady as we need to put Cilium in.

To apply Cilium we need to generate the Kubernetes manifest. We can do this by running

```bash
helm template cilium cilium/cilium --version 1.13.0 --namespace kube-system \
  --set ipam.mode=kubernetes \
  --set=kubeProxyReplacement=disabled \
  --set=securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
  --set=securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
  --set=cgroup.autoMount.enabled=false \
  --set=cgroup.hostRoot=/sys/fs/cgroup > cilium.yaml
```

Then we can apply with 

```bash
kubectl apply -f cilium.yaml
```

We also need to install CSR approver for automatic certificate approval in the cluster

```bash
INSERT TODO HERE 
```

We can watch the pods start up and check all run ok

```bash
kubectl get pods -A -w
```

Check the Kubernetes pod logs work

```bash
kubectl logs -n kube-system kube-scheduler-geb
```

Then we have a Kubernetes cluster ready to run pods, but no storage or external load balancer. 

# Cilium network configuration

We've got the Kubernetes cluster up and running but Cilium is not yet configured to announce external IPs out. 

To announce the Load Balancers we will use BGP to our router. For this setup we will have a VYOS router in the network running our routing system.