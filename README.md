# Certified Kubernetes Administration certification study

This repository contains practice commands and links to kubernetes.io website for reference to study for the CKA exam.

The curriculum of the certification can be found on the following link:
https://github.com/cncf/curriculum/blob/master/CKA_Curriculum_v1.23.pdf

## Table of contents
1. [Cluster Architecture, Installation & Configuration](1-cluster-architecture-installation-configuration.md)
1. [Workloads & Scheduling](2-workloads-scheduling.md)
1. [Services & Networking](3-services-networking.md)
1. [Storage](4-storage.md)
1. [Troubleshooting](5-troubleshooting.md)


## Helpful commands

### Create a Pod to troubleshoot cluster
```bash
kubectl run -i --tty alpine --image=alpine --restart=Never --rm -- sh
```

## .vimrc (~/.vimrc)
```bash
set tabstop=2
set shiftwidth=2
set softtabstop=2
set expandtab
set autoindent
```
