# Installing CRI-O, Kubernetes with Kubeadm, MetalLB, Contour, Calico, and Longhorn on Ubuntu Server (Plus Kubeadm cluster upgrade steps)

This doc will get you up and running with a K8s cluster on Ubuntu `minimal` server install, complete with Calcio cluster networking, Longhorn persistent storage, MetalLB load balancer, and Contour ingress controller. I've modified the tolerations for Calico controller pods so that you can run a fully functional K8s platform with just a single control plane node (It's commented in the manifest for calico. 

This setup is for a simple, single control plane node result. While it is possible to change a kubeadm deployed single cp node to HA multi-cp node cluster, it is not supported by kubeadm and is not very intuitive. For a multi control plane node cluster, read the docs on HA deployment and/or see the HA optional link in the `kubeadm init` section. 

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

## Install CRI-O on Ubuntu

**1. Set variables for subsequent commands**
```bash
export KUBERNETES_VERSION=1.31
export CRIO_VERSION=1.30
```

**2. Configure apt certs and repos**
```bash
cat <<EOF | sudo tee /etc/apt/sources.list.d/cri-o.sources
Enabled: yes
Types: deb
URIs: https://pkgs.k8s.io/addons:/cri-o:/stable:/v$CRIO_VERSION/deb/
Suites: /
Signed-By: /usr/share/keyrings/cri-o-apt-keyring.gpg
EOF
```

```bash
curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/stable:/v$CRIO_VERSION/deb/Release.key | sudo gpg --dearmor -o /usr/share/keyrings/cri-o-apt-keyring.gpg
```

**3. Update apt and install CRI-O**
```bash
sudo apt update && sudo apt -y install cri-o
```

**4. Enable and start CRI-O service**
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now crio
```

**5. Check status of service for `running`**
```bash
sudo apt install cri-tools
sudo systemctl status crio
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
curl -fsSL https://pkgs.k8s.io/core:/stable:/v$KUBERNETES_VERSION/deb/Release.key | sudo gpg --dearmor -o /usr/share/keyrings/kubernetes-key.gpg
```
```bash
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.sources
Enabled: yes
Types: deb
URIs: https://pkgs.k8s.io/core:/stable:/v$KUBERNETES_VERSION/deb/
Suites: /
Signed-By: /usr/share/keyrings/kubernetes-key.gpg
EOF
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

This will create a single node control plane. To create a multi-node control plane, replace step one below with these optional directions: [HA cp quick start guide](https://github.com/n8sOrganization/kubeadm-crio-ubu2404/blob/main/ha-cp.md).

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
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/tigera-operator.yaml
```

**2. Apply basic Calico IPIP config
```yaml
cat <<EOF | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
  namespace: tigera-operator
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
    ## The following block is to avoid an issue with interface auto-detection.
    ## If it causes issues for your installation, remove it.
    nodeAddressAutodetectionV4:
      kubernetes: NodeInternalIP
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

**2. On a fresh node with Kubeadm installed, apply the `join` command from step 1 (prepend with `sudo`).**

## Install Longhorn for peristent storage

_Check for latest version [here](https://github.com/longhorn/longhorn)_

```bash
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.2/deploy/longhorn.yaml
```

## Install MetalLB and Contour

_Note: You can use kube-vip instead of MetalLB as a Cloud Provider to manage service exposure. [See directions here](https://kube-vip.io/docs/usage/cloud-provider/)._

_Check for latest version [here](https://github.com/metallb/metallb)_

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
```

Config:
```yaml
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.253.1-192.168.253.254
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-mode
  namespace: metallb-system
EOF
```

```bash
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
```

# Upgrade Cluster Version

_Note: Backup your etcd before performing an upgrade. Alternatively, use Velero to backup your entire cluster_

### First control plane node (From first control plane node console)

**Step 1. Retrieve K8s release version and bin version, and set env vars (Ubuntu)**

Check https://github.com/kubernetes/kubernetes/releases for available releases

For bins:

```bash
apt-cache policy kubeadm
```

**Step 2. Set environment vars**

```bash
K8S_RELEASE="<Release version, e.g. v1.31.0>"
```

```bash
KUBEADM_VER="<kubeadm version, e.g. 1.31.0-00>"
```

```bash
NODE_NAME="<Node name>"
```

**Step 3. Perform upgrade plan**

```bash
sudo kubeadm upgrade plan $K8S_RELEASE
```

**Step 4. Update bins**

```bash
sudo apt-get update && sudo apt-get -y -–allow-change-held-packages install kubelet=$KUBEADM_VER kubeadm=$KUBEADM_VER kubectl=$KUBEADM_VER
```

**Step 5. Perform Cordon, drain, upgrade, and uncordon**

```bash
kubectl cordon $NODE_NAME
```

```bash
kubectl drain $NODE_NAME --ignore-daemonsets
```

```bash
sudo kubeadm upgrade apply $K8S_RELEASE
```

```bash
kubectl uncordon $NODE_NAME
```

### Subsequent control plane nodes (From each control plane node console)

**Step 1. Set environment vars**

```bash
K8S_RELEASE="<Release version, e.g. v1.31.0>"
```

```bash
KUBEADM_VER="<kubeadm version, e.g. 1.31.0-00>"
```

```bash
NODE_NAME="<Node name>"
```

**Step 2. Perform upgrade plan**

```bash
sudo kubeadm upgrade plan $K8S_RELEASE
```
**Step 3. Update bins**

```bash
sudo apt-get update && sudo apt-get -y --allow-change-held-packages install kubelet=$KUBEADM_VER kubeadm=$KUBEADM_VER kubectl=$KUBEADM_VER
```

**Step 4. Perform Cordon, drain, upgrade, and uncordon**

```bash
kubectl cordon $NODE_NAME
```

```bash
kubectl drain $NODE_NAME --ignore-daemonsets
```

```bash
sudo kubeadm upgrade node $K8S_RELEASE
```

```bash
kubectl uncordon $NODE_NAME
```

### Worker nodes (From each worker node console)

**Step 1. Set environment vars**

```bash
KUBEADM_VER="<kubeadm version, e.g. 1.31.0-00>"
```

**Step 2. Update bins**

```bash
sudo apt-get update && sudo apt-get -y --allow-change-held-packages install kubelet=$KUBEADM_VER kubeadm=$KUBEADM_VER kubectl=$KUBEADM_VER
```
