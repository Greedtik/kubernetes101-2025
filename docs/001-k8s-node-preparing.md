# Prepare VM

* จะทำ Manual ที่ master1 เพื่อให้เห็นขั้นตอน
* node ที่เหลือจะ Runscript เพื่อความรวดเร็ว
* internet bandwidth มีจำกัด อาจจะช้าได้

### Disable Swap

```bash

sudo swapoff -a
sudo rm /swap.img
sed -ri '/\sswap\s/s/^#?/#/' /etc/fstab

```

### Config VM Network Configulation

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay                #จำเป็นสำหรับ container runtime เช่น containerd หรือ Docker
br_netfilter           #ทำให้สามารถใช้ iptables กับ network bridge ได้
EOF

#โหลด modules ข้างต้นเข้าระบบทันทีโดยไม่ต้องรีบูต
sudo modprobe overlay             
sudo modprobe br_netfilter        

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1    #เปิดให้ bridge network ผ่าน iptables ได้ (จำเป็นสำหรับ network policies)
net.bridge.bridge-nf-call-ip6tables = 1    #เปิดให้ bridge network ผ่าน iptables ได้ (จำเป็นสำหรับ network policies) สำหรับ ipv6
net.ipv4.ip_forward                 = 1    #เปิดให้ระบบสามารถ forward IP ได้ (จำเป็นสำหรับ pod networking)
EOF

sudo sysctl --system

```

### Install ContainerD
 
<https://github.com/containerd/containerd/blob/main/docs/getting-started.md>


1. Set up Docker's apt repository.

```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

```

2. Install Containerd.

```bash
sudo apt install containerd.io -y
```

Chage SystemdCgroup = true 


```bash
sudo containerd config default | sudo tee /etc/containerd/config.toml

# config containerd systemdCgroup 

sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

sudo systemctl restart containerd 
sudo systemctl enable containerd

```



### Install Net-tools,Storage Client

```bash

sudo apt-get update 
sudo apt-get install -y open-iscsi nfs-common net-tools


```


### Install kubectl, kubeadm, kubelet

<https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/>




# Prepare Node ด้วย script 


### Run Script to prepare host and install CRI-O, kubelet, kubectl

```bash
curl -s https://coderepo.openlandscape.cloud/kubernetes101/kubernetes101-2025/-/raw/main/files/01-node-prepare-containerd-for-k8s-1.34.sh | bash
```

กรณีที่มี Process ค้าง ให้รอก่อน หรือ Kill : kill <id> or kill -9 <id> ไม่ก็ reboot เครื่องไปเลย 


## Navigation

* Next: [Create K8S Cluster](/doc/02-create-k8s-cluster.md)
* [Home](/README.md)

### ----- Link -----
* [Home](/README.md)
* [Prepare VM](/doc/01-prepare-vm.md)
* [Create K8S Cluster](/doc/02-create-k8s-cluster.md)
* [Install Network Plugin](/doc/03-install-network-plugin.md)
* [kube-command](/doc/04-kube-command.md)
* [Pod And Deployment](/doc/05-pod-and-deployment.md)
* [Kubernetes Service](/doc/06-kubernetes-service.md)
* [Kubernetes Ingress](/doc/07-kubernetes-ingress.md)
* [Persistent Volume](/doc/08-persistent-volume.md)
* [Secret and Configmap](/doc/09-secret-and-configmap.md)
* [Kubernetes Pod Autoscale](/doc/10-autoscale.md)
* [Helm](/doc/11-helm.md)