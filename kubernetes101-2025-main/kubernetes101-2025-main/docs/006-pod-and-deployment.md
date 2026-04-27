# POD,Deployment


## YAML

* ไฟล์ kube yaml จะประกอบด้วย key : value
* k8s yaml จะมี key เริ่มต้นหลักๆ คือ apiVersion, kind, metadata, spec

ตัวอย่าง 
```yaml
apiVersion: v1 
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
```

* สามารถเก็บได้หลายๆ yaml ใน File เดียวโดยการคั่นด้วย ---
* มีประโยชน์ในการ Infra as a Code, การสร้าง แก้ใข ลบ การจัดเก็บในที่ต่างๆ เช่น Local, git
* สามารถนำไปใช้งานต่อได้ง่าย ยกตัวอย่างเช่นการติดตั้ง ingress Controller, Cert manager ที่สามารถนำไฟล์ไป apply ได้เลย
* สามารถใช้งานได้ตั้งทั้งการ สร้าง แก้ ไข และลบ ด้วยการใช้ kubectl apply,delete,replace -f filename
* สามารถ apply ในระดับ dir, folder ได้


### ทดลอง Deploy Web App ด้วย yaml 

ตัวอย่าง web service ที่ ทำงานอยู่บน infra อื่น <https://guestbook.undyingk8s.olsdemo.com/> 


```bash 
# deploy yaml
kubectl apply -f https://coderepo.openlandscape.cloud/kubernetes101/kubernetes101-2025/-/raw/main/files/guestbook-deployment.yaml

#Check Pod, Deployment, Service, node IP
kubectl get pod,deployment,svc,node -n demo-guestbook -owide
```
* จากนั้นทดสอบเข้าเว็บผ่านหน้า Browser ผ่าน Service ที่ชื่อ frontend-external
* Web Service นี้ใช้งานได้ปกติหรือไม่


## POD 

### Create POD ด้วย YAML 

Search ว่า kubernetes pod 

<https://kubernetes.io/docs/concepts/workloads/pods/>


Create pod yaml 

```bash
vi my-nginx-pod.yaml
```

* จากนั้นใส่ yaml code
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
spec: {} 
---
apiVersion: v1 
kind: Pod
metadata:
  name: myapp  
  namespace: myapp
spec:
  containers:
  - image: nginx
    name: nginx
```
Apply yaml
```bash
kubectl apply -f my-nginx-pod.yaml
```

Check Pod 
```bash
kubectl get pod -n myapp
```

### ลองลบ pod 

```bash
kubectl delete pod myapp -n myapp
```

* เกิดอะไรขึ้น ?

## Label and Selector 

### Label 

คือการติดฉลาก ติดป้ายให้กับ k8s resource ที่เราต้องการ เพื่อใช้ระบุให้ทำอะไร เช่น 

1. svc เลือก pod ที่จะเชื่อมต่อกับ service 
2. pod เลือก run เฉพาะ node ที่มี label ว่า  site = srb

### Selector 

คือการเลือก Label มาใช้งาน เช่น svc เลือก pod ที่จะเชื่อมต่อกับ service

## ลองใส่ label ให้ Node

ลอง check label node ดูหน่อยว่าหน้าตาเป็นยังไง

```bash
kubectl get node --show-label
```

ใส่ label 

```bash
kubectl label node <node-name-worker-01> site=srb
```

Check label ด้วย get node 

```bash
kubectl get node --show-labels| grep site=srb
```

Check Label ด้วย describe node 

```bash
kubectl describe node <node-name>
```

### ทดลอง ใส่ Label และ Select label ใน k8s Service 

edit my-nginx-pod.yaml 

```bash 
vi my-nginx-pod.yaml
```

ใส่ label และเพิ่ม k8s service พร้อมทั้ง Selector label ของ pod 
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
spec: {} 
---
apiVersion: v1 
kind: Pod
metadata:
  labels:
    app: myapp
  name: myapp  
  namespace: myapp
spec:
  containers:
  - image: nginx
    name: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: myapp
spec:
  type: NodePort
  ports:
  - port: 80
  selector:
    app: myapp
```

Apply yaml
```bash
kubectl apply -f my-nginx-pod.yaml
```

Check Pod 
```bash
kubectl get pod -n myapp
```

* ทดสอบลองเข้า App ด้วย NodePort 
* ทดลองเอา Label ออกจาก pod แล้วลองสร้างใหม่ ด้วย  
```bash 
kubectl replace -f my-nginx-pod.yaml --force
```
* ทดสอบลองเข้า App ด้วย NodePort อีกครั้ง


### ลอง run pod บน node ที่มี label ที่ต้องการเท่านั้น 

จาก lab ด้านบน เรามีทดลองใส่  label ให้กับ worker node ไปแล้ว  เราจะทดลอง run pod เฉพาะบน node ที่มี label site=srb

ให้ลบ pod เดิมออกไปก่อน 
```bash
kubectl delete pod myapp -n myapp
```

edit yaml เดิม
```bash
vi my-nginx-pod.yaml
```

Add nodeSelector to yaml

```yaml
apiVersion: v1 
kind: Pod
metadata:
  labels:
    app: myapp
  name: myapp  
  namespace: myapp
spec:
  nodeSelector:
    site: srb  
  containers:
  - image: nginx
    name: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: myapp
spec:  
  type: NodePort
  ports:
  - port: 80
  selector:
    app: myapp
```

apply yaml

```bash
kubectl apply -f my-nginx-pod.yaml
```

check pod 
```bash
kubectl get pod -n myapp -owide
```


### ทดลองลบ ด้วย yaml file 

edit yaml โดยเราจะ comment ในส่วนของ namespace ไว้ให้หมด
```bash
vi my-nginx-pod.yaml
```

comment namespace 
```yaml
#apiVersion: v1
#kind: Namespace
#metadata:
#  name: myapp
#spec: {} 
---
apiVersion: v1 
kind: Pod
metadata:
  labels:
    app: myapp
  name: myapp  
  namespace: myapp
spec:
  nodeSelector:
    site: srb  
  containers:
  - image: nginx
    name: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: myapp
spec:  
  type: NodePort
  ports:
  - port: 80
  selector:
    app: myapp
```

ทดสอบลบ ด้วย yaml file 

```bash
kubectl delete -f my-nginx-pod.yaml
```

check pod,svc
```bash
kubectl get pod,svc -n myapp
```

## Deployment 

* Deployment เป็น Resource ตัวนึงใน k8s ที่มีหน้าที่ดูเเลควบคุม  Pod
* มี Resource ตัวอื่นๆ ที่ใช้สำหรับ ดูเเล ควบคุม Pod เช่น DaemonSet, StatefullSet
* workload ทั่วๆไป น่าจะประมาณ 70-80 % จะเป็น Deployment

### ทดลองสร้าง Deployment 

สร้าง deployment yaml
```bash
vi my1stdeployment.yaml
```

ใส่ yaml code 
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my1stdeployment
  namespace: myapp
  labels:
    app: my1stdeployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my1stdeployment
  template:
    metadata:
      labels:
        app: my1stdeployment
    spec:
      containers:
      - image: registry-kubly.olsdemo.com/k8s101/my1stdeployment:v1
        name: my1stdeployment
        ports:
        - containerPort: 80
          name: my1stdeployment
          protocol: TCP
#        resources:
#          requests:
#            memory: "32Mi"
#            cpu: "200m"
#          limits:
#            memory: "64Mi"
#            cpu: "250m"
#        volumeMounts:
#        - mountPath: "/vol-01"
#          name: pvc-vol-01
#        volumeMounts:
#        - name: host-path
#          mountPath: /usr/share/nginx/html
#      imagePullSecrets:
#      - name: pull-secret-name
#      volumes:
#      - name: pvc-vol-01
#        persistentVolumeClaim:
#          claimName: gluster-pvc
#      volumes:
#      - name: host-path
#        hostPath:
#          path: /home/user01/vol-01
#      nodeSelector:
#        kubernetes.io/hostname: ols-milky-k8s-worker-01
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: myapp
spec:
  type: NodePort
  ports:
  - port: 80
  selector:
    app: my1stdeployment
```

apply yaml
```bash
kubectl apply -f my1stdeployment.yaml
```

Check Pod,svc,node IP
```bash
kubectl get pod,svc,node -n myapp -owide
```

* ทดสอบลองเข้า App ด้วย NodePort 
* ทดสอบ Delete pod 

List pod 
```bash
kubectl get pod -n myapp
```

Delete pod 
```bash
kubectl delete pod [pod-random-name] -n myapp --force
```

Check Pod 
```bash
kubectl get pod -n myapp
```

* เกิดอะไรขึ้น ?

### ทดลอง Scale pod ด้วย yaml

edit deployment yaml 

```bash 
vi my1stdeployment.yaml
```

* เปลี่ยน replicas: 1  เป็น replicas: 3 

```bash
kubectl apply -f my1stdeployment.yaml
```

Check Pod 
```bash
kubectl get pod -n myapp -owide
```

## Deploy ตัว Image ส่วนตัว, Private Registry

### Create container Image

* ถ้าเครื่องมีปัญหาไม่สามารถลง Docker ได้ ให้ใช้ Google CloudShell <https://shell.cloud.google.com/>

Create dir 
```bash
mkdir myapp

cd myapp
```

Create dockerfile 
```bash
vi Dockerfile
```

Add dockerfile code 
```bash
FROM nginx

WORKDIR /usr/share/nginx/html

COPY index.html /usr/share/nginx/html
```

Create index.html file
```bash
vi index.html
```

Add html code
```html
<html>
    <title>Hello world</title>
    <body style="background-color:rgba(48, 255, 124, 0.651);">
        <center><h1>Hello world naja, My name is XXXX</h1></center>
    </body>
</html>
```


build docker 
```bash
docker buildx build --platform=linux/amd64 -t myapp .
```

Check docker images
```bash
docker images
```

Test docker images
```bash
docker run --name myapp -d -p 8080:80 myapp
```

check docker run 
```bash
docker ps -a
```

* ทดสอบ Preview ผ่านเว็บด้วย port 8080 


### push images to Private Registry

harbor Registry <https://registry-kubly.olsdemo.com/> 

login to private registry
```bash
docker login https://registry-kubly.olsdemo.com -u k8s101

password: ไปดูใน sheet นะ
```

tag image and check images
```bash
docker tag myapp:latest registry-kubly.olsdemo.com/k8s101private/[yourname]-myapp:v1

docker images
```

push image to private registry
```bash 
docker push registry-kubly.olsdemo.com/k8s101private/[yourname]-myapp:v1
```

* check images ใน harbor <https://registryhub.openlandscape.cloud>

### Create image Pull Secret

create secret 
```bash
kubectl create -n myapp secret docker-registry my-registry  \ 
--docker-server=https://registry-kubly.olsdemo.com \
--docker-username=k8s101  \
--docker-password=ไปดูใน sheet นะ
```

Check Secret 
```bash
kubectl get secret -n myapp
```


### Edit Deployment yaml

edit deployment file 
```bash
vi my1stdeployment.yaml
```

add image pull secret and new images to deployment, change ContainerPort to 80 and change svc port to 80
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my1stdeployment
  namespace: myapp
  labels:
    app: my1stdeployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my1stdeployment
  template:
    metadata:
      labels:
        app: my1stdeployment
    spec:
      containers:
      - image: registry-kubly.olsdemo.com/k8s101private/[yourname]-myapp:v1
        name: my1stdeployment
        ports:
        - containerPort: 80
          name: my1stdeployment
          protocol: TCP
#        resources:
#          requests:
#            memory: "32Mi"
#            cpu: "200m"
#          limits:
#            memory: "64Mi"
#            cpu: "250m"
#        volumeMounts:
#        - mountPath: "/vol-01"
#          name: pvc-vol-01
#        volumeMounts:
#        - name: host-path
#          mountPath: /usr/share/nginx/html
      imagePullSecrets:
      - name: my-registry
#      volumes:
#      - name: pvc-vol-01
#        persistentVolumeClaim:
#          claimName: gluster-pvc
#      volumes:
#      - name: host-path
#        hostPath:
#          path: /home/user01/vol-01
#      nodeSelector:
#        kubernetes.io/hostname: ols-milky-k8s-worker-01
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: myapp
spec:
  type: NodePort
  ports:
  - port: 80    # change to port 80
  selector:
    app: myapp

```

apply yaml
```bash
kubectl apply -f myapp-deployment.yaml
```

* check app ผ่าน NodePort 


## Exercise 

* Create Namespace Name my-deployment
* Create deployment ด้วย file yaml
* ใช้ image ของเราเองที่ใส่ไว้ใน registry
* service ( svc ) เป็น type nodeport 
* ให้ deployment มี replicas = 5 
* ให้ pod run อยู่เฉพาะ worker 02 เท่านั้น


## Navigation
* Previous: [Kube Command](/doc/04-kube-command.md)
* Next: [Kubernetes Service](/doc/06-kubernetes-service.md)
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