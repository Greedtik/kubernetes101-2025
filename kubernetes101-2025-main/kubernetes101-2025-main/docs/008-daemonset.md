# DaemonSet 


A DaemonSet ensures that all (or some) Nodes run a copy of a Pod. As nodes are added to the cluster, Pods are added to them. As nodes are removed from the cluster, those Pods are garbage collected. Deleting a DaemonSet will clean up the Pods it created. 

Some typical uses of a DaemonSet are: 

1. running a cluster storage daemon on every node
2. running a logs collection daemon on every node
3. running a node monitoring daemon on every node

## Create Simple DaemonSet YAML File

Create daemonset yaml file
```bash
vi my-daemonset.yaml
```

Add daemonset yaml code 
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: mydaemonset
  namespace: myapp
  labels:
    app: mydaemonset
spec:
  selector:
    matchLabels:
      app: mydaemonset
  template:
    metadata:
      labels:
        app: mydaemonset
    spec:
      containers:
        - name: nginx
          image: registry-kubly.olsdemo.com/k8s101/nginx:1.29.2
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "100m"
              memory: "128Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: mydaemonset
  namespace: myapp
spec:
  type: NodePort
  ports:
  - port: 80    # change to port 80
  selector:
    app: mydaemonset              
```

get daemonset
```bash
kubectl get ds -n myapp
```

Check Pod,SVC,Node 

```bash
kubectl get pod,ds,svc,node -n myapp -owide
```

### ทำไมถึงไม่มี Replicas, ไม่มี Deploy ลง  Master Node

1. DaemonSet จะ Deploy Pod ตาม Rule เนื่องจากเราไม่ได้กำหนดอะไรเลย ก็ Deploy ลง Worker ตามปกติ  แต่จะไม่ Deploy ลง Master เหมือน Pod ทั่วไป เพราะว่า Master Node มีการทำ Taint ไว้ (ค่า default ของ kubeadm)

2. ไม่มี Replicas เพราะว่า Concept คือ Deploy ลงไป 1 POD ตามจำนวน Node ที่กำหนด Rule ไว้ เช่น ให้ลง เฉพาะ work ก็จะมี worker ละ 1  ถ้ามี 100 worker ก็มี Daemonset อันนี้ 100 อัน ( 1 pod: 1 worker )

3. ถ้าอยากให้ Run บน Node ที่ต้องการ เช่น master 3 ก็ต้องลบ Rule, taint เพื่อทำให้ตรงตามเงื่อนไข หรือการเพิ่ม tolerations ลงใน POD ที่ต้องการให้ทำงานบน taint

### ทดลองให้ Run บน master3 ด้วยการ ลบ taint (ไม่แนะนำ แต่เป็นประโยชน์ในกรณีที่ต้องการใช้ k8s node น้อยๆ ให้เป็นทั้ง master & worker ใน node เดียวกัน)

ลบ taint ออกจาก master 3 
```bash
kubectl describe node k8s101-master-03
```

เราจะเห็นรายละเอียดดังนี้ 

```
root@k8s101-master-01:/home/ubuntu# kubectl describe node k8s101-master-01
Name:               k8s101-master-01
Roles:              control-plane
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=k8s101-master-01
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/control-plane=
                    node.kubernetes.io/exclude-from-external-load-balancers=
Annotations:        node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Sun, 14 Dec 2025 08:56:22 +0000
Taints:             node-role.kubernetes.io/control-plane:NoSchedule   -> ตรงนี้ คือ taint 
Unschedulable:      false
```

Remove taint จาก master 3 

```bash 
kubectl taint nodes <node-name-ที่อยากลบ-taint> node-role.kubernetes.io/control-plane:NoSchedule-
```

Node จะแจ้งสถานะ untainted

Check POD,SVC,Node อีกครั้ง 

```bash
kubectl get pod,ds,svc,node -n myapp -owide
```

Add Taint กลับให้ Master 3 

```bash
kubectl taint nodes <node-name-ที่อยากลบ-taint> node-role.kubernetes.io/control-plane:NoSchedule
```

Node จะแจ้งสถานะ tainted

Check POD,SVC,Node อีกครั้ง 

```bash
kubectl get pod,ds,svc,node -n myapp -owide
```

Delate Pod daemonset on master node 3 

```bash
kubectl delete pod mydaemonset-xxxx -n myapp
```

Check pod again 

```bash
kubectl get pod,ds,svc,node -n myapp -owide
```

### ทดลองให้ Run บน master ทุกตัวที่มี taint ด้วยการใช้ tolerations 

```
Taint (อยู่ที่ Node)
= “Node นี้ไม่อยากให้ pod มาลง”

Toleration (อยู่ที่ Pod)
= “Pod นี้บอกว่า ฉันทน taint นี้ได้”


      tolerations:
        - key: "node-role.kubernetes.io/control-plane"
          operator: "Exists"
          effect: "NoSchedule"

key: "node-role.kubernetes.io/control-plane". => ชื่อของ taint ตรงใส่เพื่อให้รู้ว่า pod จะหน้าด้านหน้าทนต่อ taint ตัวไหน          
operator: "Exists" => ไม่สน value ของ taint
effect: "NoSchedule" => ห้าม schedule pod ใหม่บน node นี้ 

ก็คือ Pod นี้ ยอมรับ node ที่มี taint แบบ NoSchedule ที่มี effect ห้าม pod มาสร้าง โดยไม่สนใจว่า ค่า (value) จะเป็นอะไร 
```

edit daemonset yaml

```bash
vi my-daemonset.yaml
```

add yaml to spec 

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: mydaemonset
  namespace: myapp
  labels:
    app: mydaemonset
spec:
  selector:
    matchLabels:
      app: mydaemonset
  template:
    metadata:
      labels:
        app: mydaemonset
    spec:
      # add toleration ตรงนี้นะ  
      tolerations:
        - key: "node-role.kubernetes.io/control-plane"
          operator: "Exists"
          effect: "NoSchedule"    
      containers:
        - name: nginx
          image: registry-kubly.olsdemo.com/k8s101/nginx:1.29.2
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "100m"
              memory: "128Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: mydaemonset
  namespace: myapp
spec:
  type: NodePort
  ports:
  - port: 80    # change to port 80
  selector:
    app: mydaemonset 
```

Apply yaml

```bash
apply -f my-daemonset.yaml
```

Check pod,svc,node again
```bash
kubectl get pod,ds,svc,node -n myapp -owide
```


### ทดลองให้ Run เฉพาะบน master ทุกตัวเท่านั้น ด้วยการใช้ tolerations และ nodeSelector

check node lable 

```bash
kubectl get node --show-labels
```

Edit yaml 

```bash
vi my-daemonset.yaml
```

Add yaml สำหรับการ เลือก label  ใต้ spec 

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: mydaemonset
  namespace: myapp
  labels:
    app: mydaemonset
spec:
  selector:
    matchLabels:
      app: mydaemonset
  template:
    metadata:
      labels:
        app: mydaemonset
    spec:
      nodeSelector:
        master-label: "master-label"
      tolerations:
        - key: "node-role.kubernetes.io/control-plane"
          operator: "Exists"
          effect: "NoSchedule"      
      containers:
        - name: nginx
          image: registry-kubly.olsdemo.com/k8s101/nginx:1.29.2
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "100m"
              memory: "128Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: mydaemonset
  namespace: myapp
spec:
  type: NodePort
  ports:
  - port: 80    # change to port 80
  selector:
    app: mydaemonset

```

apply 

```bash
kubectl apply -f my-daemonset.yaml
```

Check pod,svc,node again
```bash
kubectl get pod,ds,svc,node -n myapp -owide
```

## Exercise

1. ให้สร้าง daemonset ใหม่ ชื่อ my-ds-01
2. ให้สร้างอยู่ใน namespace my-daemonset
3. ให้ใช้ container images ที่สร้างเองและอัพไว้ใน registry แล้วจาก lab deployment
4. ให้เป็น daemonset ที่ให้ pod run แค่บน master 02 เท่านั้น


