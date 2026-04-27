# Persistent Volume 

* Persistent Volume คือการนำพื้นที่ Storage จากที่ต่างๆ มาใช้งานบน K8S เพื่อให้ POD Claim ไปใช้งาน
* มี Disk หลายประเภทให้เลือก เช่น Hostpath, NFS, GlusterFS, Ceph, Longhorn, อื่นๆ ผ่าน CSI Driver

## Prepare NFS Server 

* Login เข้า NFS VM

### install NFS Server

* ให้ config path /nfs เพื่อแชร์ไฟล์
* Search Google :   install nfs server ubuntu 22.04 
* Ref : <https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nfs-mount-on-ubuntu-22-04>

```bash

sudo apt update
sudo apt install nfs-kernel-server -y

#create dir for share path
mkdir /nfs

#change owner
sudo chown nobody:nogroup /nfs

vi /etc/exports

## add config allow client to connect
/nfs             192.168.x.x/23(rw,sync,no_root_squash,no_subtree_check)


sudo systemctl restart nfs-kernel-server

```

## Create Persistent Volume, Persistent Volume Claim

### create Persistent volume ( PV )

* ssh ไปที่ terminal vm
* create new dir /nfs/my-manual-nfs-pv for manual pv

```bash
vi manual-nfs-pv.yaml
```

add yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-nfs-pv
  labels:
    mypv: my-nfs
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual-nfs
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /nfs/my-manual-nfs-pv
    server: 192.168.234.50  # replace-with-your-nfs-server-ip
```

check pv 
```bash
kubectl get pv
```

### Create Persistent Volume Claim ( PVC )

Create New Namespace
```bash
kubectl create ns mypv
```

```bash
vi manual-nfs-pvc.yaml
```

Add yaml
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: manual-nfs-pvc
  namespace: mypv
spec:
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual-nfs
  selector:
    matchLabels:
      mypv: my-nfs
```

Chect PVC
```bash
kubectl get pvc -n mypv
```

ตัวอย่าง
```bash
NAME             STATUS   VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
manual-nfs-pvc   Bound    manual-nfs-pv   5Gi        RWX            manual-nfs     <unset>                 5s
```

## Create NFS StorageClass

* At nfs server create new dir  /nfs/nfs-storageclass

Create StorageClass yaml
```bash
vi nfs-storageclass.yaml
```

Add Yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nfs-storageclass
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storageclass
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false"
allowVolumeExpansion: true
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: nfs-storageclass
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: nfs-storageclass
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: nfs-storageclass
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: nfs-storageclass
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: nfs-storageclass
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: nfs-storageclass
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 192.168.234.50 # your ip
            - name: NFS_PATH
              value: /nfs/nfs-storageclass
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.234.50 # your ip
            path: /nfs/nfs-storageclass
```

Apply 
```bash
nfs-storageclass.yaml
```

check storageclass
```bash
kubectl get sc
```

check provisioner pod 
```bash
kubectl get pod -n nfs-storageclass
```

### Test Create PVC From Storageclass

Create PVC yaml file
```bash
vi my-sc-claim-01.yaml
```

Add Yaml
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-sc-claim-01
  namespace: mypv  
spec:
  storageClassName: nfs-storageclass
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 3Gi
```

Apply 
```bash
kubectl apply -f my-sc-claim-01.yaml
```

Check PVC in namespace mypv
```bash
kubectl get pvc -n mypv
```

* pvc status : Bound ?
* pv name random ?
* check nfs server path /nfs/nfs-storageclass  เกิดอะไรขึ้น ?

## Use Persistent Volume Claim

Create Deployment Test Volume
```bash
vi my-test-volume-deployment.yaml
```

add yaml 
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mytestvolume-deployment
  namespace: mypv
  labels:
    app: mypv
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mypv
  template:
    metadata:
      labels:
        app: mypv
    spec:
      containers:
      - image: registry-kubly.olsdemo.com/k8s101/nginx:1.29.2
        name: nginx
        ports:
        - containerPort: 80
          name: nginx-web
          protocol: TCP
        volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: vol-name
#      imagePullSecrets:
#      - name: my-registry
      volumes:
      - name: vol-name
        persistentVolumeClaim:
          claimName: volume-name
#      nodeSelector:
#        kubernetes.io/hostname: ols-milky-k8s-worker-01
---
apiVersion: v1
kind: Service
metadata:
  name: mytestvolume
  namespace: mypv
spec:
  type: NodePort
  ports:
  - port: 80
  selector:
    app: mypv
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mytestvolume-deployment
  namespace: mypv
spec:
  parentRefs:
    - name: mygateway
      namespace: envoy-gateway-system
  hostnames:
    - "mytestvolumedeploy.k8s101.longtumlab.com"
  rules:
    - backendRefs:
        - group: ""
          kind: Service
          name: mytestvolume
          port: 80
          weight: 1
      matches:
        - path:
            type: PathPrefix
            value: /
```

Apply 
```bash
kubectl apply -f my-test-volume-deployment.yaml
```

Get service info
```bash
kubectl get pod,svc,pvc,httproute -n mypv
```

* Test access Ingress เกิดอะไรขึ้น ? 
* Create file in NFS server ที่ path ของ volume

At NFS Server 
```bash
cd /nfs/nfs-storageclass/mypv-my-sc-claim-01-pvc-xxxx

vi index.html

## add HTML Code 

Test Mount PVC, By Your-name
```
* Test access Ingress อีกครั้ง เกิดอะไรขึ้น ?
* รอ TA ไปตรวจ -> เดียวตรวจจาก url ของแต่ละคน

## Exercise 

* ให้ map volume ให้กับ grafana ที่ deploy ไว้ใน lab helm
* path data ของ grafana container คือ /var/lib/grafana
* ใช้วิธีใดก็ได้ 

* 
## Navigation
* Previous: [Kubernetes Ingress](/doc/07-k8s-bashboard.md)
* Next: [Secret and ConfigMap](/doc/09-secret-and-configmap.md)
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