# Create Kubernetes Cluster 

## Config LoadBalancer : HAproxy

Install HAproxy 
```bash
sudo apt install haproxy -y

systemctl enable haproxy
```

Config HAproxy 

```bash
cp /etc/haproxy/haproxy.cfg  /etc/haproxy/haproxy.cfg-backup

vi /etc/haproxy/haproxy.cfg
```

Add HAproxy Config 
```bash
frontend k8s-api-server
     bind *:6443
     default_backend k8s-api-server
     mode tcp
     option tcplog

backend k8s-api-server
     balance source
     mode tcp
     server k8s-master-01 172.16.51.x:6443 check
     server k8s-master-02 172.16.51.x:6443 check
     server k8s-master-03 172.16.51.x:6443 check       
```

Restart HAproxy 

```bash
systemctl restart haproxy
```

## Create masternode 

### init Cluster

* ที่ Master node

```bash

kubeadm init --control-plane-endpoint=master-node-ip-or-LoadBalancer-Virtual-ip:6443 --upload-certs

```

* หลังจากติดตั้ง Conplete ให้ Copy install detail เก็บไว้ด้วย 
* Copy kubeconfig ไปไว้ใน path default เพื่อให้ใช้ kubectl จาก path ไหนก็ได้
```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

ทดสอบใช้ command
```bash
kubectl get node
```

* หาก install ไม่ผ่าน ให้ Reset ด้วยคำสั่ง  kubeadm reset  -> ตอบ yes


### Add Master Node 

* login master node2,3 
* Copy Code สำหรับการ add worker ไปวางที่ master node 2,3  จะมีข้อความประมาณนี้

```bash
You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join api.k8slab.olsdemo.com:6443 --token l8jcssdssssssssrlto2a \
        --discovery-token-ca-cert-hash sha256:1a1f9xxxxxxxxxxxxxxxxxxxxyyyyyyyyyyyyyyyyyyyyyyyy9501fa7d929 \
        --control-plane --certificate-key 9e0656ab0xxxxxxxxxxxxxxxxxxxxyyyyyyyyyyyyyyyyyyyyyyyyfa1e5ad7b10fb617
```

* ไปที่ master node เช็ค node อีกครั้ง

```bash
kubectl get node
```

### Add Worker node 

* login worker node 
* Copy Code สำหรับการ add worker ไปวางที่ worker node  จะมีข้อความประมาณนี้
```bash
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join api.k8slab.olsdemo.com:6443 --token bnxxxxxxx2.8xxxxxxxxxrk \
        --discovery-token-ca-cert-hash sha256:a06f2xxxxxxxxxxxxxxxxxxxx3ac276812cf66523fc9a041 
```
* ไปที่ master node เช็ค node อีกครั้ง

```bash
kubectl get node
```


## Navigation
* Previous: [Prepare VM](/doc/01-prepare-vm.md)
* Next: [Install Network Plugin](/doc/03-install-network-plugin.md)
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
