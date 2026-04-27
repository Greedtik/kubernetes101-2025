# Kubernetes Pod AutoScale, HorizontalPodAutoscaler

* Pod จะมี Scale ตาม Replica ที่เราได้ตั้งค่า Min pod, Max pod ไว้ โดยขึ้นอยู่กับ workload ที่เกิดขึ้น ณ เวลานั้น
* จะต้องมีการติดตั้ง Service เพื่อ Monitor Usage ด้วย

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


## Install WRK สำหรับยิงโหลด

* ติดตั้งที่เครื่อง lb, NFS Server

```bash
sudo apt-get update 
sudo apt-get install wrk -y

wrk -v
```

# ตั้งค่า AutoScale

## Create Deployment 

### Create Namespace 

```bash
kubectl create ns [your-name]-lab-autoscale
```

### Create Deployment,svc

```bash
vi lab-autoscale-deployment.yaml
```
*** Create เอง !!! 

จากนั้นให้เอา # ข้างหน้า Resource ออก จะได้หน้าตาประมาณนี้
```yaml
    spec:
      containers:
      - image: xxxxx
        name: xxxxx
        ports:
        - containerPort: 80  # change to port 80
          name: mario-web
          protocol: TCP
        resources:
          requests:
            memory: "32Mi"
            cpu: "100m"
          limits:
            memory: "64Mi"
            cpu: "200m"
```

### ตั้งค่า HorizontalPodAutoscaler ให้กับ Deployment

CPU Base

```bash
vi hpa-cpu.yaml
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-cpu
  namespace: [your-name]-lab-autoscale
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-deployment-name
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 20
```
Apply 
```bash
kubectl apply -f hpa-cpu.yaml
```

* ในกรณีที่ต้องการ scale แบบ Memory base ใช้ yaml นี้
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-memory
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 10
```

Check HPA
```bash
watch kubectl get hpa -n myapp
```

### Gen Workload 

* ที่ master node 

List svc 
```bash
kubectl get svc -n myapp
```

* ที่ VM nfs

```bash
wrk -t4 -c4000 -d500s http://node-ip:nodeport-port-number
```

* จากนั้น สังเกตผลลัพธ์ด้วย 

```bash
watch kubectl get hpa -n myapp

wathc kubectl get pod -n myapp

wathc kubectl top pod -n myapp 
```


## Navigation
* Previous: [Secret and Configmap](/doc/10-secret-and-configmap.md)
* Next: [Helm](/doc/12-helm.md)
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