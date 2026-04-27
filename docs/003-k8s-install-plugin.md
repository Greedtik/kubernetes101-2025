# Install Network Plugin 

## Cilium 

ใน Workshop นี้เราจะใช้งาน Cilium เป็น Network Plugin 

<https://cilium.io/>


### Install Cilium Cli



```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

```

### Install Cilium

```bash

cilium install

```

### Check Cilium Status 

```bash 

cilium status

```

### Check Cilium POD 

```bash

kubectl get pod -n kube-system

```

## Install metric server 

Apply yaml
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Edit Metric Server Deployment for kubelet insecure tls
```bash
kubectl edit deployment metrics-server -n kube-system
```

จากนั้น Add Code ข้างล่างลงไป
```bash
- --kubelet-insecure-tls
```

จะมีหน้าตาประมาณนี้
```yaml

    spec:
      containers:
      - args:
        - --kubelet-insecure-tls   # เราแอดอันนี้ 
        - --cert-dir=/tmp
        - --secure-port=10250
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        image: registry.k8s.io/metrics-server/metrics-server:v0.7.2
        imagePullPolicy: IfNotPresent
        livenessProbe:

```

Check Metric pod
```bash
kubectl get pod  -n kube-system
```

## Install Headlamp

Deploy Headlamp ด้วย yaml

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/headlamp/main/kubernetes-headlamp.yaml
```

* edit Haedlam Service 

List Service in namespace kube-system

```bash
kubectl get svc -n kube-system
```
Edit headlamp Service

```bash
kubectl edit svc headlamp -n kube-system
```

Chang Type ClusterIP to Type NodePort 
```yaml
spec:
  clusterIP: 10.105.178.62
  clusterIPs:
  - 10.105.178.62
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - nodePort: 30537
    port: 80
    protocol: TCP
    targetPort: 4466
  selector:
    k8s-app: headlamp
  sessionAffinity: None
  type: NodePort  # เปลี่ยนตรงนี้ !! 
```

Get Service,node Again.

```bash
kubectl get svc,node -n kube-system -owide
```

access on your web browser.

## Create Headlamp Token

Create Service Account

```bash
kubectl -n kube-system create serviceaccount headlamp-admin
```

Create ClusterRoleBinding for clsuter Admin

```bash
kubectl create clusterrolebinding headlamp-admin --serviceaccount=kube-system:headlamp-admin --clusterrole=cluster-admin
```

Create Token for Headlamp Acount

```bash
kubectl create token headlamp-admin -n kube-system
```

Use Token to loging Headlamp.


## Navigation
* Previous: [Create K8S Cluster](/doc/02-create-k8s-cluster.md)
* Next: [Kube Command](/doc/04-kube-command.md)
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