+++
author = "Lewis Green"
title = "Kubernetes Cluster Update v1.24 to v1.27"
date = "2023-04-30"
description = "Guide to upgrading Kubernetes cluster from v1.24 to v1.27"
tags = [
    "kubernetes",
]
+++

# Overview

Kubernetes cluster is currently v1.24 and we want to go to v1.27. This is a cluster that is a single control plane node and three worker nodes.

| **Node** | **Role** |
| -- | -- |
| SG-K8S-001 | Control Plane, Master Node |
| SG-K8S-002 | Worker Node |
| SG-K8S-003 | Worker Node |
| SG-K8S-004 | Worker Node |

# Steps

Steps to upgrade. You need to go to the interim major versions so 1.25, 1.26 then 1.27.

**CAVEAT I FOUND**

The GPG keys for Podman and Google had changed so I needed to run the two commands below to add the keys in

```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
curl -fsSL https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/xUbuntu_20.04/Release.key | apt-key add -
```

**Versions**

- For v1.25 we will use 1.25.9-00
- For v1.26 we will use 1.26.4-00
- For v1.27 we will use 1.27.1-00

*Repeat from here for each version*

First we need to upgrade the Control Plane on the master nodes. This does not upgrade the kubelet version yet. 

To do this we do one control plane at a time where there are multiple, but we only have one so we need to run the following scripts on the control plane node

```bash
apt update
apt-mark unhold kubeadm
apt-get install -y kubeadm=1.25.9-00
apt-mark hold kubeadm
```

Then we need to plan and apply the upgrade. On the first control plane, run ```kubeadm upgrade plan``` and check the results.

If all is good then run ```kubeadm upgrade apply v1.25.9``` on that node. 

Then for each node of the control plane just run ```kubeadm upgrade node```

Once this is done we then need to drain the nodes of the control plane one by one and then upgrade kubectl and kubelet

On a machine with access to the cluster run

```bash
kubectl drain sg-k8s-001 --ignore-daemonsets
```

Then install the new versions and restart on the node

```bash
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.25.9-00 kubectl=1.25.9-00
apt-mark hold kubelet kubectl
systemctl daemon-reload
```

We need to update CRI-O at the same time. Update the sources.list file 

```
rm /etc/apt/sources.list.d/download_opensuse_org_repositories_devel_kubic_libcontainers_stable_cri_o_1_23_xUbuntu_20_04.list
echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/1.25/xUbuntu_20.04/ /" > /etc/apt/sources.list.d/download_opensuse_org_repositories_devel_kubic_libcontainers_stable_cri_o_1_25_xUbuntu_20_04.list
```

Then update APT, and then update CRI-O

```bash
wget -qO - https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_20.04/Release.key | apt-key add -

apt update
apt install -y cri-o cri-o-runc
```

Then restart kubelet

```bash
systemctl restart kubelet
```

Then back on the PC with access to the cluster uncordon the node

```bash
kubectl uncordon sg-k8s-001
```

For each worker node do the same as above from draining the node to here.

When you run ```kubectl get nodes -o wide``` now you will see v1.25.9 on all nodes.

Repeat all steps above to go to 1.26, then to 1.27


## Notes

When we got to 1.27 there was no pods. 

```bash
nano /var/lib/kubelet/kubeadm-flags.env
```

Remove --container-runtime remote from there