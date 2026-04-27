# Kubernetes Basic Command

### kubectl 

* การใช้ kubernetes command จะขึ้นต้นด้วย kubectl 
* เครื่องที่ต้องการใช้งาน kubectl จะต้องมี file kubeconfig และเชื่อมต่อ kube api ได้ 

ติดตั้ง kubectl บนเครื่องอื่นๆ ที่ต้องการใช้งาน
```bash
sudo apt-get update -y
sudo apt-get install -y kubectl
```
* kube command สามารถใช้ชื่อย่อของ kind แต่ละตัวได้ สามารถ list ได้ด้วยคำสั่ง
```bash 
kubectl api-resources
```
ตัวอย่าง 
```bash 
NAME                               SHORTNAMES                          APIVERSION                             NAMESPACED   KIND
componentstatuses                  cs                                  v1                                     false        ComponentStatus
configmaps                         cm                                  v1                                     true         ConfigMap
endpoints                          ep                                  v1                                     true         Endpoints
events                             ev                                  v1                                     true         Event
limitranges                        limits                              v1                                     true         LimitRange
namespaces                         ns                                  v1                                     false        Namespace
nodes                              no                                  v1                                     false        Node
persistentvolumeclaims             pvc                                 v1                                     true         PersistentVolumeClaim
persistentvolumes                  pv                                  v1                                     false        PersistentVolume
pods                               po                                  v1                                     true         Pod
replicationcontrollers             rc                                  v1                                     true         ReplicationController
resourcequotas                     quota                               v1                                     true         ResourceQuota
serviceaccounts                    sa                                  v1                                     true         ServiceAccount
services                           svc                                 v1                                     true         Service
storageclasses                     sc                                  storage.k8s.io/v1                      false        StorageClass
ingresses                          ing                                 networking.k8s.io/v1                   true         Ingress
```
* kubectl สามารถทำ Auto Complete ได้ โดยการปรับแต่งเพิ่มเติม
* kubernetes Dashboard, Kubernets UI สามารถใช้ทดแทน kubectl ได้ในบางครั้ง แต่บาง Product ก็สามารถครอบคลุมการใช้งานพื้นฐานทั่วไปได้ทั้งหมด เช่น Rancher

# ทดลองใช้ Kubernetes Command

## การเรียกดูรายละเอียดต่างๆ

1. เรียกดู Kubernetes Node 
```bash
kubectl get node
```
2. เรียกดู kubernetes Node และรายละเอียด IP, OS, Version
```bash
kubectl get node -owide
``` 
3. การเรียกดูรายละเอียดของ Kubernetes Node แบบละเอียด 
```bash
kubectl describe node node-name
```
4. การเรียกดู Namespace
```bash
kubectl get namespace

kubectl get ns
```
5. การเรียกดู POD 
```bash
kubectl get pod
```
6. การเรียกดู POD ทั้งหมดใน Cluster
```bash
kubectl get pod -A
```
7. การเรียกดู POD ใน Namespace kube-system
```bash
kubectl get pod -n kube-system
```
8. การเรียกดูรายละเอียดของ pod แบบละเอียด
```bash
kubectl describe pod name-of-pod -n name-of-namespace
```

## ทดลอง Create Edit Delete

1. Create Namespace
```bash
kubectl create ns demo
```

2. Create Nginx POD in Specific Namespace
```bash
kubectl run nginx --image=nginx -n demo
```
3. Create Service For Nginx POD
```bash
kubectl expose pod nginx --port=80 --name=nginx-service -n demo
```

### ทดสอบ Access Container POD ผ่าน Service 

* Get Pod & Service 
```bash
kubectl get pod -n demo

kubectl get svc -n demo
```

ผลลัพธ์ที่ได้ 

```bash
root@ols-milky-k8slab-master-01:/home/user01# kubectl get pod -n demo
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          3h22m

root@ols-milky-k8slab-master-01:/home/user01# kubectl get svc -n demo
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
nginx-service   ClusterIP   10.106.1.246   <none>        80/TCP    4m13s
```

ทดสอบ Curl Nginx ผ่าน Service ClusterIP โดยใช้ VM ภายใน Kubernetes Cluster

```bash 
curl 10.106.1.246
```

ผลลัพธ์ที่ได้

```html
root@ols-milky-k8slab-master-01:/home/user01# curl 10.106.1.246
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### ถ้าจะเข้าผ่าน Web Browser ด้วย Com ของเรา หรือคนทั่วไปจะเข้าเว็บนี้ ?

### Edit Service ผ่าน kubectl

หลังจากที่ Get ข้อมูล หรือ ชื่อของ Service มาแล้วให้ทำการ Edit ผ่าน kubectl 

```bash
kubectl edit svc nginx-service -n demo
```

แก้ใข code ของ Service

type: ClusterIP => เปลี่ยนให้เป็น NodePort  

```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2023-10-25T07:54:24Z"
  labels:
    run: nginx
  name: nginx-service
  namespace: demo
  resourceVersion: "1458300"
  uid: 59ee0fca-c66a-4496-b6cf-287ff44dc3be
spec:
  clusterIP: 10.106.1.246
  clusterIPs:
  - 10.106.1.246
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx
  sessionAffinity: None
  type: NodePort   <<<<-- เปลี่ยนตรงนี้ !!
status:
  loadBalancer: {}

```

ระบบจะแจ้งว่า Service ถูก Edit 
```
service/nginx-service edited
```

ทดลอง Get รายละเอียดของ Service อีกครั้ง

```bash 
kubectl get svc -n demo
```

จะพบว่า TYPE = NodePort แล้ว และจะแสดงหมายเลข Port ในลักษณะ 80:30551/TCP
```
root@ols-milky-k8slab-master-01:/home/user01# kubectl get svc -n demo
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-service   NodePort   10.106.1.246   <none>        80:30551/TCP   33m

```

เรียกดูรายละเอียด IP ของ Node 

```bash 
kubectl get node -owide
```
จะพบรายละเอียดของ IP Node 

```bash
root@ols-milky-k8slab-master-01:/home/user01# kubectl get node -owide
NAME                         STATUS   ROLES           AGE    VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
ols-milky-k8slab-master-01   Ready    control-plane   5d3h   v1.28.3   192.168.234.31   <none>        Ubuntu 22.04.3 LTS   5.15.0-87-generic   cri-o://1.28.1
ols-milky-k8slab-master-02   Ready    control-plane   5d3h   v1.28.3   192.168.234.32   <none>        Ubuntu 22.04.3 LTS   5.15.0-87-generic   cri-o://1.28.1
ols-milky-k8slab-master-03   Ready    control-plane   5d3h   v1.28.3   192.168.234.33   <none>        Ubuntu 22.04.3 LTS   5.15.0-87-generic   cri-o://1.28.1
ols-milky-k8slab-worker-01   Ready    <none>          5d3h   v1.28.3   192.168.234.34   <none>        Ubuntu 22.04.3 LTS   5.15.0-87-generic   cri-o://1.28.1
ols-milky-k8slab-worker-02   Ready    <none>          5d3h   v1.28.3   192.168.234.35   <none>        Ubuntu 22.04.3 LTS   5.15.0-87-generic   cri-o://1.28.1
ols-milky-k8slab-worker-03   Ready    <none>          5d3h   v1.28.3   192.168.234.36   <none>        Ubuntu 22.04.3 LTS   5.15.0-87-generic   cri-o://1.28.1 

```

่ทดสอบ curl ด้วย ip ของ node ใดๆ ก็ได้ใน Cluster แล้วตามด้วย Port ของ NodePort 

ตัวอย่าง 

curl 192.168.234.35:30551

```html
root@ols-milky-k8slab-master-01:/home/user01# curl 192.168.234.35:30551
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### ทดสอบ Access หน้าเว็บ Nginx

ทดลองนำ 192.168.234.35:30551 ไปวางใน Web Broser 

### Exec to Pod 

use 
```bash
kubectl exec -n namespace [POD] -- [COMMAND]
```

ทดสอบใช้ ls ใน container, list ชื่อ pod ที่ต้องการก่อน 

```bash
kubectl get pod -n demo
```

ทดลอง exec 

```bash
kubectl exec -n demo nginx -- ls
```

exec เข้าไปใช้ CLI ใน container  โดยการ เพิ่ม -it และระบุ shell เช่น sh, bash, zsh, fish แล้วแต่ที่ container นั้นใช้

```bash
kubectl exec -n demo -it nginx -- bash
```

### ดู Log ของ pod

use

```bash
kubectl logs pod-name -n namespace 
```

ดุ Log Nginx POD

```bash
kubectl logs nginx -n demo
```

### ลอง Describe ดูรายละเอียดต่างๆ 

```bash
kubectl describe pod <pod-name> -n demo
```

### Delete Service, POD

* เรียกดูรายละเอียด POD และ Service

```bash
kubectl get all -n demo
```
ลบ POD และ Service

```bash
kubectl delete pod nginx -n demo

kubectl delete svc nginx-service -n demo

```

ทดสอบ Get Pod, Service ใน Namespace Demo จะพบว่าไม่มี POD, Service แล้ว

```bash
kubectl get all -n demo
```

### Delete Namespace

```bash
kubectl delete ns demo
```

## Exercise 

* Create Namespace Name kube-command
* Create Pod ด้วย kubectl command และกำหนดให้ \
  Name=mypod01 \
  image=registry-kubly.olsdemo.com/k8s101/kubecommand-lab:v1 \
  namespace=kube-command 
* Expose Pod ด้วยคำสั่งด้านล่างนี้ 
```bash
kubectl expose pod mypod01 --port=80 --name=mypod01-svc --type=NodePort -n kube-command
```
* open app in web Browser
* รอ TA ไปตรวจ !!!

## Navigation
* Previous: [Install Network Plugin](/doc/03-install-network-plugin.md)
* Next: [Pod And Deployment](/doc/05-pod-and-deployment.md)
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