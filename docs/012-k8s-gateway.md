# Gateway


## Install Enboy Gateway

Official Web <https://gateway.envoyproxy.io/>

Install Document <https://gateway.envoyproxy.io/docs/install/install-helm/>

### Prepare Helm Values

From Official git <https://github.com/envoyproxy/gateway/blob/main/charts/gateway-helm/values.tmpl.yaml>.  ใช้สำหรับการ Ref


Creat envoy dir 
```bash
mkdir envoy

cd envoy
```

Create values.yaml
```vash
vi values.yaml
```

add yaml code 

```yaml
deployment:
  replicas: 2

gateway:
  metrics:
    enabled: true
    port: 9901 
  envoy:
    gateway:
      enablePerRouteStats: true
    service:
      type: ClusterIP
    admin:
      port: 9901  # Envoy admin port
      address: 0.0.0.0
    metrics:
      enabled: true
      service:
        annotations:
          prometheus.io/scrape: "true"
          prometheus.io/port: "9901" 
```

Install envoy ด้วย helm 

```bash
helm upgrade --install eg oci://docker.io/envoyproxy/gateway-helm --version v0.0.0-latest -f values.yaml -n envoy-gateway-system --create-namespace 
```

Check pod,svc,node 
```bash
kubectl get pod,svc,node -n envoy-gateway-system -owide
```

## Create Envoy Gateway

Create SSL Certificate 

```bash
## Config DNS Server 

Register to use CloudFlare : <https://www.cloudflare.com/> 

* use your email 
* give your mail for invitation to domain management
* Create your wildcard sub-domain

1. Domain -> longtumlab.com
2. DNS -> Record
3. Add Record
4. Type = A , Name = *.[your-name]k8s , IPv4 = your Public IP from Lab IPtable.
5. Disable Proxy status to DNS Only.
6. Ping Check Your sub-domain :   asdfasdf.your-name.longtumlab.com

## Create Certificate by certbot

- install Certbot

apt install certbot -y

- Gen SSL Cert

certbot -d '*.[yourname]k8s.longtumlab.com' --manual --preferred-challenges dns certonly

```


Create TLS Secret 

```bash
kubectl create secret tls my-gateway-tls --cert=fullchain.pem --key=privkey.pem -n envoy-gateway-system
```

create gateway yaml

```bash
vi gateway.yaml
```

add yaml code 

```yaml
#### cluster Scope, not use namespace 
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: mygateway
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
  parametersRef:
    group: gateway.envoyproxy.io
    kind: EnvoyProxy
    name: mygateway
    namespace: envoy-gateway-system    
---
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: mygateway
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyDeployment:
        replicas: 2
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: mygateway
  namespace: envoy-gateway-system
spec:
  gatewayClassName: mygateway
  listeners:
    - name: http
      protocol: HTTP
      port: 80
#      allowedRoutes:
#        namespaces:
#          from: All        
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - name: my-gateway-tls
            kind: Secret
#      allowedRoutes:
#        namespaces:
#          from: Selector
#          selector:
#            matchLabels:
#              gateway: "gateway-01"
      allowedRoutes:
        namespaces:
          from: All
```

apply 

```bash
kubectl apply -f gateway.yaml
```

Check pod,svc

```bash
kubectl get pod,svc -n envoy-gateway-system
```

ต้องมี svc port 443 ตามตัวอย่างนี้

```bash
root@k8s101-master-01:/home/ubuntu/envoy# kubectl get pod,svc -n envoy-gateway-system 
NAME                                                                 READY   STATUS    RESTARTS   AGE
pod/envoy-envoy-gateway-system-mygateway-5020875a-7896f74bf9-2rqsn   2/2     Running   0          2m17s
pod/envoy-envoy-gateway-system-mygateway-5020875a-7896f74bf9-d4lzv   2/2     Running   0          2m17s
pod/envoy-envoy-gateway-system-mygateway-5020875a-7896f74bf9-wncdg   2/2     Running   0          2m17s
pod/envoy-gateway-665bbf7c49-2n4xb                                   1/1     Running   0          9m2s
pod/envoy-gateway-665bbf7c49-bwgrz                                   1/1     Running   0          9m2s
pod/envoy-gateway-665bbf7c49-khzl8                                   1/1     Running   0          9m2s

NAME                                                    TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                            AGE
service/envoy-envoy-gateway-system-mygateway-5020875a   LoadBalancer   10.104.229.234   10.151.1.48   80:30328/TCP,443:30556/TCP                         2m17s
service/envoy-gateway                                   ClusterIP      10.104.95.91     <none>        18000/TCP,18001/TCP,18002/TCP,19001/TCP,9443/TCP   9m2s
```

Go to VFW and Create DNAT and Allow WAN HTTP,HTTPS 

## Create HTTProute

create httproute yaml

```bash
vi my-httproute.yaml
```

add yaml code

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my1sthttproute
  namespace: your-namespace
spec:
  parentRefs:
    - name: mygateway
      namespace: envoy-gateway-system
  hostnames:
    - "my1stdeployment.k8s101.longtumlab.com"
  rules:
    - backendRefs:
        - group: ""
          kind: Service
          name: your-service-name
          port: 80
          weight: 1
      matches:
        - path:
            type: PathPrefix
            value: /
```

apply 

```bash
kubectl apply -f my1sthttproute.yaml
```

Check httproute 

```bash
kubectl get httproute -n your-namespace
```

* Open url in web browser


## Exercise 1

* สร้าง namespace  yourname-gatewaylab
* สร้าง Deployment โดยใช้ images ของตัวเองจาก lab ก่อนๆ 
* สร้าง httproute ให้สามารถ access web ของตัวเองผ่าน internet 
* route ที่สร้างต้อง secure !




