## For Multi-Node Control Plane

This is a single part of the process. For full process, see https://github.com/n8sOrganization/kubeadm-crio-ubu2404/blob/main/README.md

For a multi-node (HA) control plane, we need to perform some additional work. Kubeadm configures the kube config with a load balancer IP addr that will frontend all cp nodes. So we need to setup an LB and then provide that address to `kubeadm init` with the `--control-plane-endpoint` option.

There are many ways to setup an LB, but a simple no-fuss way is to usee `kube-vip`. Kube-vip will run as a static pod on our control plane nodes and assigns a common virtual IP to their interfaces. It also serves as a load balancer for them. See the [docs for complete detail](https://github.com/kube-vip/kube-vip).

The simple steps:

1. Pre-create the `/etc/kubernetes/manifests/` directory

```bash
sudo mkdir /etc/kubernetes/manifests
```

2. Create env vars for the creation of the kube-vip manifest

```bash
export KUBE_VIP_ADDR=<available IP address (e.g. 192.168.1.254)>
```

```bash
export KUBE_VIP_CIDR=<IP address cidr mask bits (e.g. 16)>
```

[Check for latest release here](https://github.com/kube-vip/kube-vip/releases)
```bash
export KUBE_VIP_RELEASE=<(e.g. v0.5.11)>
```

```bash
export HOST_INTERFACE=<your host interface (e.g. ens34)>
```

2. Create `kube-vip.yaml` manifest and place in /etc/kubernetes/manifests/

```bash
cat <<EOF | sudo tee -a /etc/kubernetes/manifests/kube-vip.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: kube-vip
  namespace: kube-system
spec:
  containers:
  - args:
    - manager
    env:
    - name: vip_arp
      value: "true"
    - name: port
      value: "6443"
    - name: vip_interface
      value: "${HOST_INTERFACE}"
    - name: vip_cidr
      value: "${KUBE_VIP_CIDR}"
    - name: cp_enable
      value: "true"
    - name: cp_namespace
      value: kube-system
    - name: vip_ddns
      value: "false"
    - name: vip_leaderelection
      value: "true"
    - name: vip_leaseduration
      value: "5"
    - name: vip_renewdeadline
      value: "3"
    - name: vip_retryperiod
      value: "1"
    - name: vip_address
      value: "${KUBE_VIP_ADDR}"
    image: ghcr.io/kube-vip/kube-vip:${KUBE_VIP_RELEASE}
    imagePullPolicy: Always
    name: kube-vip
    resources: {}
    securityContext:
      capabilities:
        add:
        - NET_ADMIN
        - NET_RAW
        - SYS_TIME
    volumeMounts:
    - mountPath: /etc/kubernetes/admin.conf
      name: kubeconfig
  hostAliases:
  - hostnames:
    - kubernetes
    ip: 127.0.0.1
  hostNetwork: true
  volumes:
  - hostPath:
      path: /etc/kubernetes/admin.conf
    name: kubeconfig
EOF
```
 
 3. Init control plane

```bash
sudo kubeadm init --pod-network-cidr=10.50.0.0/16 --service-cidr=10.100.0.0/16  --cri-socket='unix:///var/run/crio/crio.sock' --control-plane-endpoint $KUBE_VIP_ADDR --upload-certs
```

4. Repeat steps one and two on subsequent control plane nodes and then use the control plane node `join` command resulting from step 3.

If it has been moore than two hours since your first control plane node was created, issue the following command on it before adding your new node:

```bash
kubeadm init phase upload-certs
```

[Return to full guide](https://github.com/n8sOrganization/kubeadm-crio-ubu2404/blob/main/README.md#init-control-plane)
 
