# Updated 2026. Install K8s with Kubeadm and Cilium

## Prep node:

```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gpg
```

## Disable swap

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## Load required kernel modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

## Set required sysctl params

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

## Install and configure containerd

```bash
sudo apt update
sudo apt install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null

sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

## Install kubeadm, kubelet, kubectl

```bash
#Pick the right version to match kubeadm version

KUBERNETES_VERSION=v1.36

sudo mkdir -p /etc/apt/keyrings

curl -fsSL "https://pkgs.k8s.io/core:/stable:/${KUBERNETES_VERSION}/deb/Release.key" \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${KUBERNETES_VERSION}/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Init Control Plane

This will create a single node control plane. To create a multi-node control plane, replace step one below with these optional directions: [HA cp quick start guide](https://github.com/n8sOrganization/kubeadm-crio-ubu2404/blob/main/ha-cp.md).

## Init cluster 

```bash
sudo kubeadm init \
  --service-cidr=172.16.0.0/12 \
  --skip-phases=addon/kube-proxy
```

## Configure kubectl

```bash
mkdir -p "$HOME/.kube"
sudo cp -i /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
```

## Check kubectl connection to cluster

```bash
kubectl get nodes
```

## Install the Cilium CLI 

```bash
CILIUM_CLI_VERSION="$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)"
CLI_ARCH="amd64"

if [ "$(uname -m)" = "aarch64" ]; then
  CLI_ARCH="arm64"
fi

curl -L --fail --remote-name-all \
  "https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz" \
  "https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz.sha256sum"

sha256sum --check "cilium-linux-${CLI_ARCH}.tar.gz.sha256sum"

sudo tar xzvf "cilium-linux-${CLI_ARCH}.tar.gz" -C /usr/local/bin

rm "cilium-linux-${CLI_ARCH}.tar.gz" "cilium-linux-${CLI_ARCH}.tar.gz.sha256sum"
```

## Check Cilium install

```bash
cilium version --client
```

## Install Cilium with kube-proxy replacement and custom PodCIDR

```bash
#Do kubectl get nodes -o wide to find your control plane node IP
API_SERVER_IP="<CONTROL_PLANE_NODE_IP>"
API_SERVER_PORT="6443"
POD_CIDR="10.0.0.0/8"

cilium install \
  --version 1.19.4 \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost="${API_SERVER_IP}" \
  --set k8sServicePort="${API_SERVER_PORT}" \
  --set ipam.mode=cluster-pool \
  --set ipam.operator.clusterPoolIPv4PodCIDRList="{${POD_CIDR}}" \
  --set ipam.operator.clusterPoolIPv4MaskSize=24
```

## Wait for Cilium to become ready

```bash
cilium status --wait
```

## Now show the join command and use it on another Ubuntun node prepared as above to join as worker node

## Join a Worker Node

To complete your cluster, repeat the disable swap, enable ip forward, add br_netfilter module, and the installation steps for Kubeadm on a fresh Linux host. Then submit the `join` command on that host.

## From the control plane node**
```bash
sudo kubeadm token create --print-join-command
```

## On a fresh node with Kubeadm installed, apply the `join` command from step 1 (prepend with `sudo`).**

For storage, use Longhorn

## Install Longhorn for peristent storage

_Check for latest version [here](https://github.com/longhorn/longhorn)_

```bash
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.4.0/deploy/longhorn.yaml
```


# Upgrade Cluster Version

_Note: Backup your etcd before performing an upgrade. Alternatively, use Velero to backup your entire cluster_

### First control plane node (From first control plane node console)

**Step 1. Retrieve K8s release version and bin version, and set env vars (Ubuntu)**

Check https://github.com/kubernetes/kubernetes/releases for available releases

For bins:

```bash
apt-cache policy kubeadm | grep <version>
```

**Step 2. Set environment vars**

```bash
K8S_RELEASE="<Release version, e.g. v1.26.2>"
```

```bash
KUBEADM_VER="<kubeadm version, e.g. 1.26.2-00>"
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
K8S_RELEASE="<Release version, e.g. v1.26.2>"
```

```bash
KUBEADM_VER="<kubeadm version, e.g. 1.26.2-00>"
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
sudo apt-get update && sudo apt-get -y –allow-change-held-packages install kubelet=$KUBEADM_VER kubeadm=$KUBEADM_VER kubectl=$KUBEADM_VER
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
KUBEADM_VER="<kubeadm version, e.g. 1.26.2-00>"
```

**Step 2. Update bins**

```bash
sudo apt-get update && sudo apt-get -y –allow-change-held-packages install kubelet=$KUBEADM_VER kubeadm=$KUBEADM_VER kubectl=$KUBEADM_VER
```
