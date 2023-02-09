# Installing CRI-O and Kubeadm v1.26.1 on Ubuntu Server v22.04

This doc will get you up and running with a K8s cluster on Ubuntu 22.04 `minimal` server install, complete with Calcio cluster networking. I've modified the tolerations for Calico controller pods so that you can run a fully functional K8s platform with just a single control plane node (It's commented in the manifest for calico. This is obviously not something you'd do outside of a lab)

With a single node, you will end up with something like this:
![image](https://user-images.githubusercontent.com/45366367/216838964-10ad77e5-fc9e-4bd8-8e77-4ffc93c8958c.png)

## Configure Ubuntu for Kubeadm and CRI-O (These steps are generally required for any CRI runtime and/or K8s)

**1. Update base ubuntu install**
```bash
sudo apt update && sudo apt -y upgrade
```

**2. Install utilities required for subsequent steps**
```bash
sudo apt install -y apt-transport-https ca-certificates curl gnupg2 software-properties-common
```

**3. Disable swap**
```bash
sudo swapoff -a
```

**4. Remark out the swap line in the fstab file and save change**
```bash
sudo vi  /etc/fstab 
```

**5. Enable ip forwarding**
```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

**6. Add `net.ipv4.ip_forward = 1` to presistent config**
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF
```

**7. Load br_netfilter and overlay module**
```bash
sudo modprobe br_netfilter
```

```bash
sudo modprobe overlay
```

**8. Add `br_netfilter` and `overlay` to persistent config**

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

## Install CRI-O on Ubuntu 22.04

**1. Set variables for subsequent commands. OS and VERSION are specific to CRI-O URLs**
```bash
export OS=xUbuntu_22.04
export VERSION=1.26
```

**2. Configure apt certs and repos**
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

**3. Update apt and install CRI-O and CRI-O specific runC**
```bash
sudo apt update && sudo apt -y install cri-o cri-o-runc
```

**4. Enable and start CRI-O service**
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now crio
```

**5. Check status of service for `running`**
```bash
sudo systemctl status crio
sudo crio-status info
sudo crictl info
```

**6. Remove a CNI directory that CRI-O creates, but we don't need**
```bash
sudo rm -rf /etc/cni
```

## Install kubeadm

**1. Update apt-get**
```bash
sudo apt-get update
```

**2. Install utils for apt-get commands**
```bash
sudo apt-get install -y apt-transport-https ca-certificates curl
```

**3. Configure apt-get cert and repo**
```bash
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```
```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

**4. Update apt and install `kubelet`, `kubeadm`, and `kubectl`**
```bash
sudo apt-get update && sudo apt-get -y install kubelet kubeadm kubectl
```

**5. Pre-fetch Kubeadm images**
```bash
sudo kubeadm config images pull
```

## Init Control Plane

**1. Change CIDRs to whatever makes sense for your environment. Will be using IPIP overlay, so as long as they don't overlap with each other or other advertised CIDRs, you are good**
```bash
sudo kubeadm init --pod-network-cidr=10.50.0.0/16 --service-cidr=10.100.0.0/16	 --cri-socket='unix:///var/run/crio/crio.sock'
```

**2. Copy kubeconfig file to $HOME/.kube**
```bash
mkdir $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown -R $(id -u):$(id -g) $HOME/.kube/config
```

## Install Calico CNI plugin with basic IPIP overlay

**1. Install Calcio operator**
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/tigera-operator.yaml
```

**2. Apply basic Calico IPIP config**
```yaml
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

**1. From the control plane node**
```bash
sudo kubeadm token create --print-join-command
```
**2. On a fresh node with Kubeadm installed, appy the `join` command from step 1.**

## Misc. Info

### BGP not working / calico-node pods not starting

If you switch between IPIP mode and VXLAN mode, and/or if you disable and re-enable BGP, you might experience a symptom of the calico-node pods not being able to re-peer with the other nodes for BGP route sharing. The default config for calico-node to determine which interface to select on a host for underlay traffic is first-detected. This typically works on a freshly istalled host. But on one that has many interfaces defined (e.g. one that has been running a CNI plugin with all sorts of veth interfaces), it can become problematic.

To fix this, you can configure the service to use a different selection method. All methods are (https://docs.tigera.io/calico/3.25/reference/configure-calico-node#ip-autodetection-methods)[documented here]. 

To fix the issue, you can try to apply this change:

```bash
kubectl set env daemonset/calico-node -n calico-system IP_AUTODETECTION_METHOD=kubernetes-internal-ip]
```

To make it permanent (when using the Calico operator), update your `installation` resource as foolows:

```yaml
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
  ...
    nodeAddressAutodetectionV4:
      kubernetes-internal-ip: true
```
