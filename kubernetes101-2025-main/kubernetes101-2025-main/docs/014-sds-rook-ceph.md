# software defined storage : Rook Ceph

Ref : <https://rook.io/>

## Install Rook Operator

```bash 
mkdir rook

cd rook
```

### install Operator

Download values 
```bash
wget https://coderepo.openlandscape.cloud/kubernetes101/kubernetes101-2025/-/raw/main/files/ceph-operator-values.yaml
```

run helm install

```bash
# add repo
helm repo add rook-release https://charts.rook.io/release

# install Operator pod
helm install --create-namespace --namespace rook-ceph rook-ceph rook-release/rook-ceph -f ceph-operator-values.yaml
```

Check pod 

```bash
kubectl get pod,svc -n rook-ceph
```

### install Cluster 

* have raw disk in worker node ?

Download values 
```bash
wget https://coderepo.openlandscape.cloud/kubernetes101/kubernetes101-2025/-/raw/main/files/ceph-cluster-values.yaml
```

Check values

```bash
vi ceph-cluster-values.yaml
```


### run helm install Cluster 

```bash
helm upgrade --install -n rook-ceph rook-ceph-cluster --set operatorNamespace=rook-ceph rook-release/rook-ceph-cluster -f ceph-cluster-values.yaml
```

*** ถ้ามี error ในการ upgrade อาจจะต้อง force ด้วย --force-conflicts

Check Pod,SVC

```bash
kubectl get pod,svc -n rook-ceph
```


## Check Rook Ui and create httproute

```bash
vi cephui.yaml
```

add yaml code 

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: cephui
  namespace: rook-ceph
spec:
  parentRefs:
    - name: mygateway
      namespace: envoy-gateway-system
  hostnames:
    - "cephui.[yourname]k8s.longtumlab.com"
  rules:
    - backendRefs:
        - group: ""
          kind: Service
          name: rook-ceph-mgr-dashboard
          port: 8443
          weight: 1
      matches:
        - path:
            type: PathPrefix
            value: /
```

Apply yaml
```bash
kubectl apply -f cephui.yaml
```

Check Httproute

```bash
kubectl get pod,svc,httproute -n rook-ceph
```

## Check Storage Class 

```bash
kubectl get sc 
```

## Check Ceph ObjectStore

```bash
kubectl get cephobjectstore -A
```