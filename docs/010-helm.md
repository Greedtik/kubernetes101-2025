# Helm 

Helm คือ Package managerment บน k8s ที่จะช่วยให้จัดการ application ต่างๆ ได้ง่ายขึ้น โดยที่เราไม่ต้องเขียน yaml เอง

## Install Helm on Master Node

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
```

Check Helm Version 
```bash
helm version
```

## Using Helm : Deploy Grafana 

Ref, Official Doc <https://grafana.com/docs/grafana/latest/setup-grafana/installation/helm/>

### add Helm Repo 
```bash
helm repo add grafana https://grafana.github.io/helm-charts
```

### List Helm Repo

```bash
helm repo list
```

### Update Repo 
```bash
helm repo update
```

### Search Repo

```bash
helm search repo <repo-name>

helm search repo grafana
```

### run Grafana

```bash
kubectl create ns my-helmlab

helm install <name> <repo>/<chart-name> --set option  -n <name-space>

helm install grafana grafana/grafana --set service.type=NodePort  -n my-helmlab

# List helm ที่ได้ Deploy ไปแล้ว

helm list -n my-helmlab

Helm status name -n namespace

#check service
kubectl get pod,deployment,svc -n my-helmlab
```

ทดสอบ access nodeport เพื่อ Check Service 


### Uninstall Grafana

List helm 

```bash
helm list -A
```

Uninstall Helm App

```bash
helm uninstall grafana -n my-helmlab
```

List helm 
```bash
helm list -A
```

Check service 
```bash
kubectl get pod,deployment,svc -n my-helmlab
```

## Helm Values

Helm Values เป็น Yaml File Config ที่จะช่วยให้การ Config รายละเอียดต่างๆ ง่ายขึ้น 

### Get Helm Values from helmchart 
```bash
helm show values <repo/chart-name>

helm show values grafana/grafana

## save values as file
helm show values <repo/chart-name> > values.yaml

## exaple 
helm show values grafana/grafana > grafana-values.yaml
```

Edit Helm Values
```bash
vi grafana-values.yaml
```

change yaml code

```yaml
#Line 233
##
service:
  enabled: true
  type: ClusterIP -> change to NodePort
  # Set the ip family policy to configure dual-stack see [Configure dual-stack](https://kubernetes.io/docs/concepts/services-networking/dual-stack/#services)
  ipFamilyPolicy: ""

```

Install Grafana ด้วย helm values

```bash
helm install grafana grafana/grafana -f grafana-values.yaml  -n my-helmlab
```

Get App Service 
```bash
kubectl get pod,deployment,svc -n my-helmlab -owide
```


## Upgrade Grafana ด้วย Helm Values

label worker node 03

```bash
kubectl label node your-worker-03 node=node03
```

edit grafan values.yaml
```bash
vi grafana-values.yaml
```

Change Values
```yaml
#Specific Node 
# Get node label by " kubectl get node --show-labels "
# edit values for Deploy app to worker 3,
#Line 365

#Before 
nodeSelector: {}

#After
# depend on your node label

nodeSelector:
  kubernetes.io/hostname: your-worker-node-03-label
```

Run Helm upgrade

```bash
helm upgrade grafana grafana/grafana -f grafana-values.yaml -n my-helmlab
```

เราสามารถใช้ command เดียวในการ install หรือ upgrade ได้ ด้วย 

```bash
helm upgrade --install grafana grafana/grafana -f grafana-values.yaml -n my-helmlab
```

## Navigation
* Previous: [Secret and Configmap](/doc/10-secret-and-configmap.md)
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