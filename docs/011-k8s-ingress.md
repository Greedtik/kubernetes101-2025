# k8s ingress








## Install Traefik Ingress Controller 


### Create Traefik Namespace

```bash
kubectl create namespace traefik

kubectl get ns
```

### Add Traefik's Helm Repo 

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
```
### Create Traefik helm values.yaml

Follow this doc <https://doc.traefik.io/traefik/setup/kubernetes/>

```bash
vi values.yaml
```

Add yaml ตามเอกสารด้านบน และ add helm values ด้านล้างนี้เพิ่มเข้าไป  

```yaml
deployment:
  replicas: 2

service:
  type: NodePort
  nodePorts:
    http: 30080 # Optional: Specify a custom NodePort for the 'web' entry point
    https: 30443 # Optional: Specify a custom NodePort for the 'websecure' entry point

```

### Install Traefik ingress Controller 

```bash
helm upgrade --install traefik traefik/traefik   --namespace traefik   --values values.yaml
```

### Check Service 

Check Traefik pod Replicas = 2 and Service Type = NodePort

```bash
kubectl get pod,svc -n traefik
```

ตัวอย่างผลลัพธ์

```bash
root@k8s101-master-01:/home/ubuntu/traefik# kubectl get pod,svc -n traefik
NAME                           READY   STATUS    RESTARTS   AGE
pod/traefik-78cb666d56-dhpns   1/1     Running   0          18m
pod/traefik-78cb666d56-gkgpn   1/1     Running   0          11m

NAME              TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/traefik   NodePort   10.100.124.105   <none>        80:30080/TCP,443:30443/TCP   18m

```

### Config HAproxy

Install HAproxy 
```bash
sudo apt install haproxy -y

systemctl enable haproxy
```

Config HAproxy 

```bash
vi /etc/haproxy/haproxy.cfg
```

Add HAproxy Config เพิ่มเข้าไป 

```bash       
frontend app-80
     bind *:80
     default_backend app-80
     mode tcp
     option tcplog

backend app-80
     balance source
     mode tcp
     server k8s-worker-01 172.16.51.x:80 check
     server k8s-worker-02 172.16.51.x:80 check
     server k8s-worker-03 172.16.51.x:80 check

frontend app-443
     bind *:443
     default_backend app-443
     mode tcp
     option tcplog

backend app-443
     balance source
     mode tcp
     server k8s-worker-01 172.16.51.x:443 check
     server k8s-worker-02 172.16.51.x:443 check
     server k8s-worker-03 172.16.51.x:433 check
```

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

## Test Ingress 

Create Ingress YAML 

```bash
vi my-ingress.yaml
```
add ingress yaml

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: your-ingress-name
  namespace: your-namespace
  annotations:
    #cert-manager.io/cluster-issuer: your-clusterissuer-name
    #nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    #nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    # type of authentication
    #nginx.ingress.kubernetes.io/auth-type: basic
    # name of the secret that contains the user/password definitions
    #nginx.ingress.kubernetes.io/auth-secret: basic-auth
    # message to display with an appropriate context why the authentication is required
    #nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required - foo'
spec:
  ingressClassName: class-name 
  #tls:
  #- hosts:
  #    - test-ingress.k8s101v1.olsdemo.com
  #  secretName: test-ingress
  rules:
  - host: test-ingress.k8s101v1.olsdemo.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: demo
            port:
              number: 80

```

Apply ingress 

```bash
kubectl apply -f my-ingress.yaml
```

Check ingress 

```bash
kubectl get pod,svc,ing -n your-namespace 
```

Test Access Ingress Hostname in your web browser.

## Secure Ingress

install Certbot
```bash
apt install certbot -y
```

Gen SSL Cert
```bash 

certbot -d '*.[yourname]k8s.longtumlab.com' --manual --preferred-challenges dns certonly
```

Create TLS Secret 

```bash
kubectl create secret tls my-tls --cert=fullchain.pem --key=privkey.pem -n my-namespace
```

edit ingress yaml

```bash
vi my-ingress.yaml
```

edit yaml

```yaml
spec:
  ingressClassName: class-name 
  #tls:
  #- hosts:
  #    - test-ingress.k8s101v1.olsdemo.com
  #  secretName: test-ingress <<< Secret name ตรงนี้ๆๆๆ
  rules:
  - host: test-ingress.k8s101v1.olsdemo.com
    http:
      paths:
      - path: /
        pathType: Prefix
```



# Cert Manager 


## Install Cert manager 

Officail Web : <https://cert-manager.io/>

* Document -> Select version to 1.17 
* Installation -> kubectl apply
* Install from the cert-manager release manifest


Check Cert manager POD

```bash
kubectl get pods --namespace cert-manager
```

## Config ClusterIssuer 

create cluster issuer file 

```bash
vi clusterissuer.yaml
```

add yaml code 

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: clusterissuer-name
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@gmail.com
    privateKeySecretRef:
      name: clusterissuer-name
    solvers:
      - http01:
          ingress:
            class: your-ingress-class-or-traefik
```

apply cluster issuer file 

```bash
kubectl apply clusterissuer.yaml
```

Check Cluster Issuer 

```bash
kubectl get clusterissuer
```

ถ้า Cluster Issuer พร้อมใช้งาน จะต้องมี status True : ตัวอย่าง 

```bash
root@k8s101-master-01:/home/ubuntu/certmanager# kubectl get clusterissuer
NAME              READY   AGE
clusterissuer01   True    4d23h
envoy01           True    2d11h
```

## Config ingress 

```bash
vi my-ingress.yaml
```
Edit Annotation ตรง cluster issuer 

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: your-ingress-name
  namespace: your-namespace
  annotations:
    #cert-manager.io/cluster-issuer: [your-clusterissuer-name]  << ตรงนี้ ๆๆๆๆ
    #nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    #nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
```

Apply 

```bash
kubectl apply -f my-ingress.yaml
```

Check pod,svc,ing

```bash
kubectl get pod,svc,ing,certificate -n your-namespace 
```

* ทดสอบ access web ผ่าน ui อีกครั้ง
* หากมีปัญหา ลองเปิดในโหมด incognito ก่อน


## Exercise 1

1. ให้สร้าง deployment ใหม่ ชื่อ my-deploy-and-ing
2. ให้สร้างอยู่ใน namespace lab-ingress
3. ให้ใช้ container images ที่สร้างเองและอัพไว้ใน registry แล้วจาก lab deployment
4. ให้มี replicas = 6 
5. ห้ามให้ pod run อยู่บน Node เดียวกัน ( node ละ 1 pod เท่านั้น)
6. สร้าง ingress ด้วยชื่อ my1stingress.[yourname]k8s.longtumlab.com
7. ให้ ingress ที่สร้างมา มีความ secure โดยการใช้ certmanager 

## Exercise 2

ให้สร้าง secured ingress ให้กับ  headlamp ui

