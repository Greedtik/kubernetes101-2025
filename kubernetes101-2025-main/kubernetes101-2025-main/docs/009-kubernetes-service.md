# Kubernetes Service 

* NodePort 
* ClusterIP 
* LoadBalancer

## Service Type NodePort 

* จาก lab ที่ผ่านๆ มาน่าจะเห็นแนวคิดการใช้งาน NodePort
* ใช้ host network เป็นตัวเชื่อมต่อลงไปยัง Container Network ที่อยู่ใน Cluster ด้วยการ Assign Port Network ในช่วง 30000 - 32767 ( Default ) 


## Service Type ClusterIP

* Service ClusterIP จะเป็น Service ที่อยู่ภายใน Cluster สำหรับให้ Service ข้างใน Cluster เชื่อมต่อกันเท่านั้น คล้ายๆ Private Network VXLAN บน Hypervisor 
* การ Connect อะไรก็ตามใน k8s จะใช้ชื่อ Service แทน IP เนื่องจากชื่อจะไม่มีเปลี่ยนแปลง แต่ ip จะมีการสุ่มสร้าง และ random ไปเรื่อยๆ 
* Service ClusterIP จะเป็น Service ที่ User ที่อยู่ภายนอก Cluster ไม่สามารถเข้าถึงได้ จะต้องทำ Forward Port หรือ Ingress เข้าไปเชื่อมต่อ

### Deploy POD Connnection Tester

create yaml
```bash
vi nettools-deployment.yaml
```
add yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nettools
  namespace: myapp
  labels:
    app: nettools
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nettools
  template:
    metadata:
      labels:
        app: nettools
    spec:
      containers:
      - image: registry-kubly.olsdemo.com/k8s101/nettools:latest
        name: nettools
        command: ["sleep", "3600"]
        ports:
        - containerPort: 8080
          name: nettools
          protocol: TCP 
---
apiVersion: v1
kind: Service
metadata:
  name: nettools
  namespace: myapp
spec:
  type: NodePort
  ports:
  - port: 8080
  selector:
    app: nettools
```

Apply
```bash
#create pod
kubectl apply -f nettools-deployment.yaml

#check pod and service 
kubectl get pod,svc -n myapp
```


* exec เข้า pod nettools หรือใช้ UI headlamp ก็ได้ 

```bash
kubectl exec -n myapp -it network-tools-xxxxxx -- bash
```

* ทดสอบผ่าน curl ไปยัง service ปลายทางที่ต้องการ

```bash
curl service-name port
```

Connect ข้าม namespace 

```bash
#curl ไปหา guestbook
curl frontend.demo-guestbook.svc.cluster.local

#curl ไปหา nginx lab ใน demo
curl nginx-service.demo.svc.cluster.local

# Ncat ทดสอบ TCP 9153 ของ Core DNS
nc -v kube-dns.kube-system.svc.cluster.local 9153
```

## Service Type LoadBalancer 

* Service Type LoadBalancer ในระบบ Public Cloud จะทำการ Config LB ให้และ แจก IP ที่ผู้ใช้สามารถ Access ได้จริงๆ ให้ สามารถนำ IP นั้นไปใช้งานได้ทันที
* ในกรณีที่ระบบ Cloud ไม่สามารถใช้งาน Type LoadBalancer ได้ Service จะมีสถานะ Pending
* หากต้องการใช้งาน Service Type LoadBalancer จะต้องมี LB ที่สามารถทำงานร่วมกับ k8s ได้ หรือใช้ Software ช่วย เช่น MetalLB หรือ OpenELB, etc.


## ทดสอบลองใช้งาน Service Type LoadBalancer

edit my1stdeployment.yaml 

```bash
vi my1stdeployment.yaml
```

Chang Service Type NodePort to LoadBalancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: myapp
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: my1stdeployment
```

Apply 

```bash
kubectl apply -f my1stdeployment.yaml
```

Check SVC status 

```bash
kubectl get svc -n myapp
```

* เกิดอะไรขึ้น ?


## Install MetalLB

ref: <https://metallb.universe.tf/installation/>

```bash 
#install metalLB 
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml 

# check pod
kubectl get pod -n metallb-system
```

Config IP Pools
```bash
vi metallb-ip-pool.yaml
```
add yaml
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: my-pools
  namespace: metallb-system
spec:
  addresses:
  - 10.151.0.xx-10.151.0.xx
```

Apply yaml
```bash
kubectl apply -f metallb-ip-pool.yaml
```

config L2Advertisement

```bash
vi metallb-l2advertisement.yaml
```

add yaml
```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
    - my-pools
```

Apply yaml
```bash
kubectl apply -f metallb-l2advertisement.yaml
```

* หลังจาก ติดตั้งเสร็จ service ของ my1stdeployment จะมีการแจก ip ให้ จากที่เคยมีสถานะ pending สามารถทดสอบเข้าใช้งานผ่าน ip ได้

list google microservice Service 
```bash
kubectl get svc -n myapp
```
* จากนั้นทดสอบเข้าเว็บผ่านหน้า Browser ด้วย IP ที่ได้จาก Service Type LoadBalancer.


## Navigation
* Previous: [Pod And Deployment](/doc/05-pod-and-deployment.md)
* Next: [Kubernetes Ingress](/doc/07-kubernetes-ingress.md)
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