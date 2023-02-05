# Installing CRI-O for Kubeadm v1.26.1 on Ubuntu Server v22.04

This doc will get you up and running with a K8s cluster on Ubuntu 22.04, complete with Calcio cluster networking. I've modified the tolerations for Calico controller pods so that you can run a fully functional K8s platform with just a single controlplane node (It's commented in the manifest for calico. This is obviously not something you'd do outside of a lab)

With a single node, you will end up with something like this:
![image](https://user-images.githubusercontent.com/45366367/216838964-10ad77e5-fc9e-4bd8-8e77-4ffc93c8958c.png)

## Configure Linux for Kubeadm and CRI-O

*Update base install*
```bash
sudo apt update && upgrade
```

*Install utilities required for subsequent steps*
```bash
sudo apt install -y apt-transport-https ca-certificates curl gnupg2 software-properties-common
```

*Disable swap*
```bash
sudo swapoff -a
```

*Remark out the swap line in the fstab file and save change*
```bash
sudo vi  /etc/fstab 
```

*Enable ip forwarding* 
```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

*Add `net.ipv4.ip_forward = 1` to end of sysctl.conf*
```bash
sudo vi /etc/sysctl.conf
```

*Load br_netfilter module*
```bash
sudo modprobe br_netfilter
```

*Add `br_netfilter` to last line of /etc/modules*
```bash
sudo vi /etc/modules
```

## Install CRI-O on Ubuntu 22.04

*Set variables for subsequent commands. OS and VERSION are specific to CRI-O URLs*
```bash
export OS=xUbuntu_22.04
export VERSION=1.26
```

*Configure apt certs and repos*
```bash
echo "deb [signed-by=/usr/share/keyrings/libcontainers-archive-keyring.gpg] https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
```

```bash
echo "deb [signed-by=/usr/share/keyrings/libcontainers-crio-archive-keyring.gpg] https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /" | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list
```

```bash
curl -fsSL https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | sudo gpg --dearmor -o /usr/share/keyrings/libcontainers-archive-keyring.gpg
```

```bash
curl -fsSL https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/Release.key | sudo gpg --dearmor -o /usr/share/keyrings/libcontainers-crio-archive-keyring.gpg
```

*Update apt and install CRI-O and CRI-O specific runC*
```bash
sudo apt update && sudo apt install -y cri-o cri-o-runc
```

*Enable and start CRI-O service*
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now crio
```

*Check status of service for `running`*
```bash
sudo systemctl status crio
sudo crio-status info
sudo crictl info
```

*Remove a CNI directory that CRI-O shouldn't have created*
```bash
sudo rm -rf /etc/cni
```

## Install kubeadm

*Update apt-get*
```bash
sudo apt-get update
```

*Install utils for apt-get commands
```bash
sudo apt-get install -y apt-transport-https ca-certificates curl
```

*Configure apt-get cert and repo*
```bash
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```
```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

*Update apt and install `kubelet`, `kubeadm`, and `kubectl`*
```bash
sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
```

*Pre-fetch Kubeadm images*
```bash
sudo kubeadm config images pull
```

## Init Controlplane

*Change CIDRs to whatever makes sense for your environment. Will be using IPIP overlay, so as long as they don't overlap with each other or other advertised CIDRs, you are good*
```bash
sudo kubeadm init --pod-network-cidr=10.50.0.0/16 --service-cidr=10.100.0.0/16	 --cri-socket='unix:///var/run/crio/crio.sock'
```

*Copy kubeconfig file to $HOME/.kube*
```bash
mkdir $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown -R $(id -u):$(id -g) $HOME/.kube/config
```

## Install Calico CNI plugin with basic IPIP overlay

*Install Calcio operator*
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/tigera-operator.yaml
```

*Apply basic Calico IPIP config*
```bash
cat <<EOF | kubectl create -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 10.50.0.0/16
      encapsulation: IPIP
      natOutgoing: Enabled
      nodeSelector: all()
  ## The following block is only added so pods will tolerate 
  ## controlplane nodes. Not normal. If you plan to add
  ## a worker node, it can be removed.
  controlPlaneTolerations:
    - key: "node-role.kubernetes.io/control-plane"
    - key: "node-role.kubernetes.io/master"
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF
```

## Join a Worker Node

To complete your cluster, repeat the disable swap, enable ip forward, add br_netfilter module, and the installation steps for Kubeadm on a fresh Linux host. Then submit the `join` command on that host.

*From the controlplanenode*
```bash
sudo kubeadm token create --print-join-command
```


