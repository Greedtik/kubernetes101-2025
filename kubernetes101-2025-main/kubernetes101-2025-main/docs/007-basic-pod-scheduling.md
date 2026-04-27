# Pod Scheduling

In Kubernetes, scheduling refers to making sure that Pods are matched to Nodes so that Kubelet can run them.


## Node Selector

- Use Node Label
- Node Selector on pod, Deployment

Labels Node
```bash
kubectl label node node-name zone=bkk
```

Add Selector on Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  ....
spec:
  ....
    spec:
      containers:
      - image: xxx
#      nodeSelector:
#        label-key: label-values
      nodeSelector:
        kubernetes.io/hostname: ols-milky-k8s-worker-01
```

* Apply yaml file 
* Check pod

## Node Affinity

ให้ POD ไปอยู่ Node, Zone ที่ต้องการ 

```bash
ประเภท

requiredDuringSchedulingIgnoredDuringExecution
→ ต้องตรง ไม่งั้น Pod จะ Pending

preferredDuringSchedulingIgnoredDuringExecution
→ พยายามเลือก ถ้าไม่ได้ก็ไป Node อื่นได้
```

edit your deployment

```vi 
my1stdeployment.yaml
```

Add yaml Code 
```yaml

spec:
  ...  
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: nodetype
                operator: In
                values:
                - kodfast                  
```

Apply 

```bash
kubectl apply -f my1stdeployment.yaml
```

Check Pod 

```bash
kubectl get pod,node -n myapp -owide
```

เกิดอะไรขึ้น ?

Add Node Labels
```bash
kubectl label node your-k8s-worker-03 nodetype=kodfast
```

Check pod again 

```bash
kubectl get pod,node -n myapp -owide
```

### ทดสอบ Change required to preferred

edit your deployment 

```bash
vi my1stdeployment.yaml
```

Edit Yaml
```yaml
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: nodetype
                operator: In
                values:
                - kodfast 
```

Apply 

```bash
kubectl apply -f my1stdeployment.yaml
```

Check Pod 

```bash
kubectl get pod,node -n myapp -owide
```

เกิดอะไรขึ้น ?

### Test Edit Label

remove labels nodetype=kodfast
```bash
kubectl label node k8s101-worker-03 nodetype-
```

restart pod
```bash
kubectl delete pod my1stdeployment-xxxxx -n myapp
```


Check Pod 

```bash
kubectl get pod,node -n myapp -owide
```

เกิดอะไรขึ้น ?


## Pod Affinity

- กำหนดให้ Pod นี้อยากอยู่ใกล้ Pod อื่น (เช่น อยู่ Node เดียวกัน / zone เดียวกัน)


edit yaml 

```bash
vi my1stdeployment.yaml
```

edit yaml code 

```yaml

  replicas: 6
    ....

      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - my1stdeployment
            topologyKey: kubernetes.io/hostname
```

apply and delete pod 

```bash
kubectl apply -f my1stdeployment.yaml

kubectl delete --all pod -n myapp --force
```

Check Pod 

```bash
kubectl get pod,node -n myapp -owide
```

เกิดอะไรขึ้น ?

## Pod Anti-Affinity

Pod นี้ ห้าม อยู่ใกล้ Pod อื่น เพื่อไม่ให้ down ไปพร้อมกัน

edit yaml 

```bash
vi my1stdeployment.yaml
```

edit yaml code 

```yaml
replicas:2
  ...

      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - my1stdeployment
            topologyKey: kubernetes.io/hostname

```

apply 

```bash
kubectl apply -f my1stdeployment.yaml
```
Check Pod 

```bash
kubectl get pod,node -n myapp -owide
```

test delete เพื่อดูการกระจายตัวใหม่
```bash
kubectl delete --all pod -n myapp --force
```

## taint

Taint ใน Kubernetes คือกลไกที่ใช้ กัน (ป้องกัน) ไม่ให้ Pod ไปรันบน Node บางตัว เว้นแต่ Pod นั้นจะ “ยอมรับได้” โดยการใส่ Toleration

```bash
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

Check Master Node taint

```bash
kubectl describe node master-node-name
```

edit yaml

```bash
vi my1stdeployment.yaml
```

add yaml

```yaml

replicas:6

    ...
 
      tolerations:
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
```

apply and check pod

```bash
kubectl apply -f my1stdeployment.yaml

kubectl get pod,node -n myapp -owide
```