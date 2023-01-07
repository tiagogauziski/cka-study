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

### .vimrc (~/.vimrc)
```bash
set tabstop=2
set shiftwidth=2
set softtabstop=2
set expandtab
set autoindent
```


### Linux `journalctl` usage:
```bash
# Tail logs from kubelet (live updates) 
# -f or --follow 
journalctl -u kubelet -f

# Jump to the end of the log
# -e or --page-end
journalctl -u kubelet -e

# Show the most recent number of *n* number of log lines
# -n or --lines
journalctl -u kubelet -n 10
```

### Install containerd manually
```bash
# Download containerd binaries
# AMD64
wget https://github.com/containerd/containerd/releases/download/v1.6.15/containerd-1.6.15-linux-amd64.tar.gz

# ARM64
wget https://github.com/containerd/containerd/releases/download/v1.6.15/containerd-1.6.15-linux-arm64.tar.gz

# Unpack into /usr/local/bin/ (change the arch in case of ARM64)
sudo tar Cxzvf /usr/local containerd-1.6.8-linux-amd64.tar.gz

# Download runc command line tool
# AMD64
wget https://github.com/opencontainers/runc/releases/download/v1.1.4/runc.amd64
# ARM64
wget https://github.com/opencontainers/runc/releases/download/v1.1.4/runc.arm64

# Install runc (change the arch in case of ARM64)
sudo install -m 755 runc.amd64 /usr/local/sbin/runc

# Download cni binaries
# AMD64
wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
# ARM64
wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-arm64-v1.1.1.tgz

# Create a new directory for cni binaries
sudo mkdir -p /opt/cni/bin

# Unpack cni binaries (change the arch in case of ARM64)
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz

# Create directory for containerd configuration
sudo mkdir /etc/containerd

# Create default configuration
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup 
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

# Download systemd file
sudo curl -L https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -o /etc/systemd/system/containerd.service

# Reload systemd deamon
sudo systemctl daemon-reload

# Enable containerd service
sudo systemctl enable --now containerd

# Verify containerd service status
sudo systemctl status containerd
```