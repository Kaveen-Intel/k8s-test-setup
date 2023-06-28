**K8s Cluster Installation Guide**

- [Disclaimer](#disclaimer)
  - [Versioning](#versioning)
  - [Tools to install K8s](#tools-to-install-k8s)
  - [Machine Prerequisites](#machine-prerequisites)
  - [Where to run commands](#where-to-run-commands)
- [Container runtime](#container-runtime)
  - [Overview](#overview)
  - [Installation](#installation)
    - [Steps applicable to all container runtimes](#steps-applicable-to-all-container-runtimes)
    - [Steps applicable to `containerd`](#steps-applicable-to-containerd)
- [K8s Components](#k8s-components)
- [Install cluster](#install-cluster)
  - [Prior considerations](#prior-considerations)
  - [Initialize cluster (Run only in masters)](#initialize-cluster-run-only-in-masters)
  - [Pod network add-on](#pod-network-add-on)
  - [Joining worker nodes (Only run in workers)](#joining-worker-nodes-only-run-in-workers)
  - [Label roles of worker nodes (Only run in masters)](#label-roles-of-worker-nodes-only-run-in-masters)
- [Install Helm (wherever you want it to be)](#install-helm-wherever-you-want-it-to-be)
- [Install K8s Dashboard](#install-k8s-dashboard)
- [Install Prometheus (This is very manual, implement dynamic provisioning!)](#install-prometheus-this-is-very-manual-implement-dynamic-provisioning)
- [Install Grafana](#install-grafana)
- [Linking Prometheus to Grafana](#linking-prometheus-to-grafana)

## Disclaimer

### Versioning

- At the point of writing, during the installation process, version-related flags was **not** specified in commands so the latest versions were installed by default
  - **Specify** those flags for reproducibility in subsequent environment

<br>

### Tools to install K8s

- `kubeadm` was used for the installation
- It is recommended, however to use tools like `kubeSpray`, `okps` for K8s clusters in production environment since they automate management infrastructure-related tasks as well

<br>

### Machine Prerequisites

**Computing resources**

- A compatible Linux host. The K8s project provides generic instructions for Linux distributions based on Debian and Red Hat, and those distributions without a package manager.
- 2 GB or more of RAM per machine (any less will leave little room for your apps).
- 2 CPUs or more.

**Networking**

- Full network connectivity between all machines in the cluster (public or private network is fine).
- Unique hostname, MAC address, and product_uuid for every node. See here for more details.
  - `hostnamectl` to configure hostname. This is also benefical for readability when `ssh`-ing, etc
  - `ip link` or `ifconfig -a` to verify unique MAC
  - `sudo cat /sys/class/dmi/id/product_uuid` to verify unique product_uuid
- Certain ports are open on your machines.
  - See [here](https://kubernetes.io/docs/reference/networking/ports-and-protocols/) for the required ports
  - Use `netcat` to check if port is open. Eg: `nc 127.0.0.1 6443`

**Memory optimization**

- Swap disabled. You **MUST** disable swap in order for the kubelet to work properly.
  - `sudo swapoff -a` will disable swapping temporarily. To make this change persistent across reboots, make sure swap is disabled in config files like /etc/fstab, systemd.swap, depending how it was configured on your system.

<br>

### Where to run commands

Execute on all nodes unless otherwise stated

## Container runtime

### Overview

- To run containers in Pods, Kubernetes uses a container runtime.
- By default, K8s uses the Container Runtime Interface (`CRI`) to interface with your chosen container runtime.
- We'll use `containerd`. Another popular option is `CRI-O`
  - Note that `containerd` is only `CRI`-compliant because it uses a cri plugin.
  - `CRI-O` is an implementation of `CRI` itself
  - Therefore, `CRI-O` might be the better alternative in the long run

<br>

### Installation

#### Steps applicable to all container runtimes

1. Forwarding IPv4 and letting iptables see bridged traffic

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

2. Verify that the `br_netfilter`, `overlay` modules are loaded

```
lsmod | grep br_netfilter
lsmod | grep overlay
```

3. Verify that the `net.bridge.bridge-nf-call-iptables`, `net.bridge.bridge-nf-call-ip6tables`, and `net.ipv4.ip_forward` system variables are set to `1`

```
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

<br>

#### Steps applicable to `containerd`

1. Uninstall all conflicting packages
   ```
   for pkg in containerd runc; do sudo apt-get remove $pkg; done
   ```
2. Update the `apt` package index and install packages to allow `apt` to use a repository over HTTPS
   ```
   sudo apt-get update
   sudo apt-get install ca-certificates curl gnupg
   ```
3. Add Docker’s official GPG key
   ```
   sudo install -m 0755 -d /etc/apt/keyrings
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
   sudo chmod a+r /etc/apt/keyrings/docker.gpg
   ```
4. Set up the repository
   ```
   echo \
   "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
   "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   ```
5. Install `containerd`
   ```
   sudo apt-get install containerd.io
   ```
6. Update `/etc/containerd/config.toml`
   ```
   containerd config default > /etc/containerd/config.toml
   ```
7. Address the following issues
   1. Installing `containerd` via package managers will disable the cri plugin it uses to interface with `CRI`. We need to enable it.  
      **FIX**
   ```
   Replace line with `disabled_plugins ...` to `enabled_plugins = ["cri"]`
   ```
   2. Need to change the cgroup driver to `systemd` if Linux distrubution is using `systemd` as `init system`. Check what `init system` you're using by doing `man init` .  
      **NOTE**: `kubelet` and `container runtime` should use same cgroup driver + changing the cgroup driver of a Node that has joined a cluster is sensitive since `kubelet` already created pods using other cgroup driver  
      **FIX**
   ```
   Replace line with `SystemdCgroup ...` to `SystemdCgroup = true`
   ```
   3. There's a [open bug](https://github.com/cri-o/cri-o/issues/6985) in K8s `1.27` where `kubeadm` pulls `pause:3.6` instead of `pause:3.9` as the `CRI` overrides it. This causes issues later on.  
      **FIX**
   ```
   Replace `sandbox_image = "registry.k8s.io/...` to `sandbox_image = "registry.k8s.io/pause:3.9`
   ```
8. Restart `containerd`
   ```
   sudo systemctl restart containerd
   ```

## K8s Components

- Knowledge of version skews is essential if upgrading/you want to install specific versions.
  - Refer [here](https://kubernetes.io/releases/version-skew-policy/) for version skew policies betweeen K8s components
  - Refer [here](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#kubeadm-s-skew-against-the-kubelet) for version skew for `kubeadm`

1. Update and install packages needed for K8s repository
   ```
   sudo apt-get update
   sudo apt-get install -y apt-transport-https ca-certificates curl
   ```
2. Download public signing key
   ```
   curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
   ```
3. Add K8s repository
   ```
   echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
   ```
4. Download K8s components
   ```
   sudo apt-get update
   sudo apt-get install -y kubelet kubeadm kubectl
   sudo apt-mark hold kubelet kubeadm kubectl
   ```

## Install cluster

### Prior considerations

1. If you have plans to upgrade this single control-plane `kubeadm` cluster to high availability you should specify the `--control-plane-endpoint` to set the shared endpoint for all control-plane nodes to either a DNS name/load-balancer's IP
2. Depending on the Pod network add-on, you might need to set the `--pod-network-cidr` (range of IPs to assign to pods in nodes in their network namespace) to a specific value
   1. I'm using `Calico`. By default, `192.168.0.0/16` CIDR is used. If it overlaps with host network, change it
   2. In my case, my host network was `192.168.0.0.21` (ie overlapped with default CIDR) so I changed it to `10.244.0.0/16`
3. Specify `--cri-socket` if there're multiple `CRI`s in node
4. Specify `--apiserver-advertise-addresss=<ip-address>` if you do not want `kubeadm` to use the default value (ie network interface related to default gateway)

<br>

### Initialize cluster (Run only in masters)

1. Initialize (cluster was single master thus `--control-plane-endpoint` is omitted)
   ```
   sudo kubeadm init --apiserver-advertise-addresss=<ip-address> --pod-network-cidr=10.244.0.0/16
   ```
2. Authenticate
   ```
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```
3. Check cluster by checking status of pods via
   `kubectl get po -A -o wide` or `kubectl get --raw='/readyz?verbose'`

<br>

### Pod network add-on

1. Install calico operator
   ```
   kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
   ```
2. Pull custom resources to configure Calico CNI itself
   ```
   curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml -O
   ```
3. Edit manifest optionally (I needed to change the `spec.calicoNetwork.ipPools[].cidr` from `192.168.0.0/16` to `10.244.0.0/16` )
4. Install manifest
   ```
   kubectl create -f custom-resources.yaml
   ```
   <br>

### Joining worker nodes (Only run in workers)

1. Run this
   ```
   kubeadm token create --print-join-command
   ```
2. Copy and paste the output and run it

<br>

### Label roles of worker nodes (Only run in masters)

1. For each worker node, do this:
   ```
   kubectl label node <worker-node-name>  node-role.kubernetes.io/worker=worker
   ```
2. Check node status with `kubectl get no`

## Install Helm (wherever you want it to be)
  ```
  curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
  sudo apt-get install apt-transport-https --yes
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
  sudo apt-get update
  sudo apt-get install helm
  ```

<br>

## Install K8s Dashboard

1. Install repository
   ```
   helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
   ```
2. Create custom values file
   ```
   service:
     type: "NodePort"
     nodePort: 31000

   metrics-server:
     enabled: true
     defaultArgs:
       - --kubelet-insecure-tls=true
       - --cert-dir=/tmp
       - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
       - --kubelet-use-node-status-port
       - --metric-resolution=15s

   metricsScraper:
   enabled: true: true
   ```
3. Install chart (It's good practice to save to a local file and then `kubectl apply -f`. We'll skip it for brevity)
   ```
   helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard -f dashboard-values.yaml
   ```
4. Generate RBAC resources (Please configure the permissions appropriately. For simplicity, we'll assign `cluster-admin` privileges)
   ```
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: super-dashboard-user
     namespace: monitoring
   
   ---
   
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: super-dashboard-user
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: cluster-admin
   subjects:
   - kind: ServiceAccount
     name: super-dashboard-user
     namespace: monitoring
   ```
5. Generate token and paste output inside the field that you will see on `https://<nodeIp>:<nodePort>`
   ```
   kubectl -n monitoring create token super-dashboard-user
   ```
   ![Dashboard-login-screen](image-1.png)

<br>

## Install Prometheus (This is very manual, implement dynamic provisioning!)
1. Install repository
   ```
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   ```
2. Create necessary `PersistentVolume` in `test-worker-01`
   1. Since we're using `local` volumes, we need to create some folders beforehand
   2. Need to also create `StorageClass` despite `local` volumes not supporting dynamic provisioning becasue we want to delay volume binding till Pod is in `Scheduled` state.
   ```
   sudo mkdir -p /mnt/pv/prometheus-server
   sudo mkdir -p /mnt/pv/prometheus-alertmanager
   sudo chmod 755 -R /mnt/pv
   ```
   <br>

   ```
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: local-storage
   provisioner: kubernetes.io/no-provisioner
   volumeBindingMode: WaitForFirstConsumer

   ---

   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: local-storage-prometheus-server
   spec:
     capacity:
       storage: 5Gi
     volumeMode: Filesystem # Mount volume into Pod as a directory.
     accessModes:
     - ReadWriteOnce
     persistentVolumeReclaimPolicy: Retain
     storageClassName: local-storage
     local:
       path: /mnt/pv/prometheus-server # Path to the directory this PV refers to.
     nodeAffinity: # nodeAffinity is required when using local volumes.
       required:
         nodeSelectorTerms:
         - matchExpressions:
           - key: kubernetes.io/hostname
             operator: In
             values:
             - test-worker-01
   ---

   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: local-storage-prometheus-alertmanager
   spec:
     capacity:
       storage: 2Gi
     volumeMode: Filesystem # Mount volume into Pod as a directory.
     accessModes:
     - ReadWriteOnce
     persistentVolumeReclaimPolicy: Retain
     storageClassName: local-storage
     local:
       path: /mnt/pv/prometheus-alertmanager # Path to the directory this PV refers to.
     nodeAffinity: # nodeAffinity is required when using local volumes.
       required:
         nodeSelectorTerms:
         - matchExpressions:
           - key: kubernetes.io/hostname
             operator: In
             values:
             - test-worker-01
   ```
3. Create `PVC` for `prometheus-server`
   ```
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: prometheus-server-pvc
     namespace: monitoring
   spec:
     accessModes:
       - ReadWriteOnce
     storageClassName: local-storage
     resources:
       requests:
         storage: 5Gi
   ```
4. Install the manifests (the `StorageClass`, `PVC`, `PV`) and helm charts
   ```
   kubectl apply -f <the above manifests>

   helm install prometheus prometheus-community/prometheus \
    --set server.persistentVolume.existingClaim=prometheus-server-pvc \
    --set server.persistentVolume.size=5Gi \
    --set server.persistentVolume.storageClass=local-storage \
    --set alertmanager.persistence.storageClass=local-storage \
    -n monitoring 
   ```
5. Expose it
   ```
   kubectl expose service prometheus-server --type=NodePort --target-port=9090 -n monitoring --name=prometheus-server-ext
   ```

<br>

## Install Grafana

1. Install repository
   ```
   helm repo add grafana https://grafana.github.io/helm-charts
   helm repo update
   ```
2. Create necessary `PersistentVolume` in `test-worker-02`
   1. Since we're using `local` volumes, we need to create some folders beforehand
   2. Same `StorageClass` (ie `local-storage`) is used
   ```
   sudo mkdir -p /mnt/pv/grafana
   sudo chmod 755 -R /mnt/pv
   ```
   <br>

   ```
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: local-storage-grafana
   spec:
     capacity:
       storage: 10Gi
     volumeMode: Filesystem # Mount volume into Pod as a directory.
     accessModes:
     - ReadWriteOnce
     persistentVolumeReclaimPolicy: Retain
     storageClassName: local-storage
     local:
       path: /mnt/pv/grafana # Path to the directory this PV refers to.
     nodeAffinity: # nodeAffinity is required when using local volumes.
       required:
         nodeSelectorTerms:
         - matchExpressions:
           - key: kubernetes.io/hostname
             operator: In
             values:
             - test-worker-02
   ```
3. Create `PVC` for `grafana`
   ```
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: grafana-pvc
     namespace: monitoring
   spec:
     accessModes:
       - ReadWriteOnce
     storageClassName: local-storage
     resources:
       requests:
         storage: 10Gi
   ```
4.  Install the manifests (the  `PVC`, `PV`) and helm charts   
    ```
    kubectl apply -f <the above manifests>

    helm install grafana grafana/grafana \
     --set persistence.enabled=true \
     --set persistence.size=10Gi \
     --set persistence.existingClaim=grafana-pvc \
     --set persistence.storageClassName=local-storage \
     -n monitoring
    ```
5. Expose it (Check `grafana` secret for the username/passoword)
   ```
   kubectl expose service grafana --type=NodePort --target-port=3000 -n monitoring --name=grafana-ext
   ```

## Linking Prometheus to Grafana
1. Refer [here](https://prometheus.io/docs/visualization/grafana/#creating-a-prometheus-data-source)
2. Create dashboard by clicking on `Dashboards` in sidebar > `New` on the right side > `Import`
   1. Specify a dashboard id, I went for `315`. It looks like this
      ![Alt text](image-2.png)