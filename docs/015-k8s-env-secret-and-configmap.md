
# ENV, Secret and ConfigMap

## Environment Variables

Ref : <https://12factor.net/>

* ตัวแปรที่ถูกตั้งค่าในระบบปฏิบัติการหรือในสภาพแวดล้อมการทำงานของโปรแกรม 
* ไม่ต้องเขียน Code ใหม่
* ใช้ส่งค่า config, Secret 

## ทดลองใช้ ENV

Create ns 

```bash
kubectl create ns [your-name]-envlab
```

Create deployment yaml
```bash
vi my-env-lab.yaml
```

Add yaml Code

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: yourname-envlab
  namespace: yourname-envlab
  labels:
    app: envlab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: envlab
  template:
    metadata:
      labels:
        app: envlab
    spec:     
      containers:
      - image: registry-kubly.olsdemo.com/k8s101/envlab:v1
        name: envlab
        ports:
        - containerPort: 80
          name: envlab
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
#      - name: my-registry 
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
  name: yourname-envlab
  namespace: yourname-envlab
spec:
  type: NodePort
  ports:
  - port: 80
  selector:
    app: envlab
```

Apply

```bash
kubectl apply -f my-env-lab.yaml
```

Access application 

### Test Add ENV 

Add More YAML Code
```yaml
# add ภายใต้ Container 

        env:
        - name: BG_COLOR
          value: "green"
        - name: MY_MESSAGE
          value: "My Name Is xxx"
```

Apply

```bash
kubectl apply -f my-env-lab.yaml
```

จากนั้น access ผ่าน Web Browser , ผลเป็นอย่างไร ?


# Secret 
* เป็น Resource สำหรับเก็บข้อมูลที่เป็นความลับ แทนที่เราจะแสดงข้อมูลนั้นใน yaml,config ตรงๆ 

## ทดลองใช้ Secret 

Deploy minio app

```bash
vi minio-deployment.yaml
```

add yaml

```yaml
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: minio-pvc
  namespace: yourname-envlab
spec:
  storageClassName: nfs-storageclass
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio-01
  namespace: yourname-envlab
  labels:
    app: minio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      hostname: "minio-api"
      containers:
      - image: quay.io/minio/minio:RELEASE.2023-11-01T01-57-10Z-cpuv1 
        name: minio
        ports:
        - containerPort: 9000
          name: web-port
          protocol: TCP
        command:
           - /bin/bash
           - '-c'
        env:
          - name: MINIO_ROOT_USER
            value: admin
          - name: MINIO_ROOT_PASSWORD  
            value: OLSworkshop2025      
          - name: MINIO_SERVER_URL
            value: "https://minio-api.user[x].olsdemo.com:443"       
          - name: CONSOLE_SECURE_TLS_REDIRECT
            value: "off"
          - name: MINIO_BROWSER_REDIRECT_URL
            value: "https://minio-console.user[x].olsdemo.com"   
          - name: MINIO_PROMETHEUS_AUTH_TYPE
            value: "public"              
        args:
          - 'minio server /data --console-address :9090'
        volumeMounts:
        - mountPath: /data
          name: minio-vol
      volumes:
      - name: minio-vol
        persistentVolumeClaim:
          claimName: minio-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: minio-console
  namespace: yourname-envlab
spec:
  type: NodePort
  ports:
  - port: 9090
    targetPort: 9090
  selector:
    app: minio
---
apiVersion: v1
kind: Service
metadata:
  name: minio-api
  namespace: yourname-envlab
spec:
  type: ClusterIP
  ports:
  - port: 9000
    targetPort: 9000
  selector:
    app: minio
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minio-console
  namespace: minio
  annotations:
    cert-manager.io/cluster-issuer: clusterissuer01
    nginx.ingress.kubernetes.io/proxy-body-size: '0'
spec:
  ingressClassName: nginx
  tls:
  - hosts:
      - minio-console.user[x].olsdemo.com
    secretName: minio-console-tls
  rules:
  - host: minio-console.user[x].olsdemo.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: minio-console
            port:
              number: 9090
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minio-api
  namespace: minio
  annotations:
    cert-manager.io/cluster-issuer: clusterissuer01
    nginx.ingress.kubernetes.io/proxy-body-size: '0'
spec:
  ingressClassName: nginx
  tls:
  - hosts:
      - minio-api.user[x].olsdemo.com
    secretName: minio-api-tls
  rules:
  - host: minio-api.user[x].olsdemo.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: minio-api
            port:
              number: 9000              
```

Get info
```bash
kubectl get pod,svc,pvc,ing -n minio
```

* ทดสอบเข้า minio ผ่าน minio-console.user[x].olsdemo.com
* login เข้า minio
* Review YAML

## Create Secret for minio

create secret yaml file
```bash
vi minio-secret.yaml
```

Add yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: minio-secret
  namespace: minio
type: kubernetes.io/basic-auth
stringData:
  username: admin # required field for kubernetes.io/basic-auth
  password: OLSworkshop2025  # required field for kubernetes.io/basic-auth
```

Apply 
```bash
kubectl apply -f minio-secret.yaml
```

Check Secret 
```bash
kubectl get secret -n minio
```

### Edit minio deployment 

```bash
vi minio-deployment.yaml
```

Edit yaml
```yaml
#เปลี่ยน
        env:
          - name: MINIO_ROOT_USER
            value: admin
          - name: MINIO_ROOT_PASSWORD  
            value: OLSworkshop2025 

#เป็น
        env:
          - name: MINIO_ROOT_USER
            valueFrom:
              secretKeyRef:
                name: minio-secret
                key: username 
          - name: MINIO_ROOT_PASSWORD  
            valueFrom:
              secretKeyRef:
                name: minio-secret
                key: password  
```

Apply 
```bash
kubectl apply -f minio-deployment.yaml
```

* ทดสอบเข้า minio ผ่าน minio-console.user[x].olsdemo.com อีกครั้ง
* login เข้า minio ยังเข้าได้ด้วย username,pass เหมือนก่อน edit หรือไม่
* Review YAML


## Create Secret From Your Certificate

* ในการทำงานจริง บริษัทจะมี Certificate ให้ใช้งาน เราจะนำ Cert นั้นมาใช้งานอย่างไร ? 

Download your Certificate : <https://olslab-fileserver.undyingk8s.olsdemo.com/>


### Create Secret 

* สร้าง Secret จาก Commandline จะสะดวกกว่าการสร้างด้วย yaml เพราะเพียงแค่ชี้พาร์ทไปที่ชื่อไฟล์เท่านั้น ไม่ต้องเอาข้อมูลลงไปวางใน yaml 
* ทดสอบเข้า minio ผ่าน minio-console.user[x].olsdemo.com อีกครั้ง แล้วสังเกต cert ข้อมูลวันที่สร้าง,วันหมดอายุ, อื่นๆ 

วางทีละบรรทัด

```bash
kubectl create secret tls my-ssl -n minio \
  --cert=fullchain.pem \
  --key=privkey.pem
```

Check Secret 
```bash
kubectl get secret -n minio

## ลอง check ต้นทาง & ลองแกะ secret ดู

kubectl get secret my-ssl -n minio -o yaml

echo "your-cert-data" | base64 -d
```

Edit Deployment minio
```bash
vi minio-deployment.yaml
```

edit deployment ของ ingress 
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minio-console
  namespace: minio
  annotations:
    #cert-manager.io/cluster-issuer: clusterissuer01   ### ปิดไว้
    nginx.ingress.kubernetes.io/proxy-body-size: '0'
spec:
  ingressClassName: nginx
  tls:
  - hosts:
      - minio-console.user[x].olsdemo.com
    secretName: my-ssl   ### your secret name, this lab is my-ssl
  rules:
  - host: minio-console.user[x].olsdemo.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: minio-console
            port:
              number: 9090
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minio-api
  namespace: minio
  annotations:
    #cert-manager.io/cluster-issuer: clusterissuer01  ### ปิดไว้
    nginx.ingress.kubernetes.io/proxy-body-size: '0'
spec:
  ingressClassName: nginx
  tls:
  - hosts:
      - minio-api.user[x].olsdemo.com
    secretName: my-ssl    ### your secret name, this lab is my-ssl
  rules:
  - host: minio-api.user[x].olsdemo.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: minio-api
            port:
              number: 9000
```

* ทดสอบเข้า minio ผ่าน minio-console.user[x].olsdemo.com อีกครั้ง แล้วสังเกต cert ข้อมูลวันที่สร้าง,วันหมดอายุ, อื่นๆ 




# Configmap

Configmap เป็น Resource บน k8s ที่จะช่วยให้เราปรับแต่งแก้ไขค่า config ที่อยู่ใน Container ได้โดยที่เราไม่ต้องแก้ไขไฟล์นั้นแล้ว Build เป็น image ใหม่ หรือต้อง mount ไฟล์จาก storage เข้าไปเพื่อแก้ไข เนื่องจากข้อมูลบน container จะไม่คงอยู่ถาวร

## Create HA Proxy Pod

Create namespace 

```bash
kubectl create ns [your-name]-configmap
```

Create HAproxy yaml

```bash
vi myconfigmap.yaml
```

```yaml

############### HAproxy Service (SVC) ############################ 
---
apiVersion: v1
kind: Service
metadata:
  name: [your-name]-haproxy
  namespace: [your-name]-configmap
  labels:
    app: haproxy
spec:
  selector:
    app: haproxy
  #type: ClusterIP
  type: ClusterIP
  ports:
    - name: http
      port: 80
      targetPort: 80
    - name: stats
      port: 8404
      targetPort: 8404
    - name: metrics
      port: 9101
      targetPort: 9101

################### HAproxy Deployment ###################################
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: [your-name]-haproxy
  namespace: [your-name]-configmap
  labels:
    app: haproxy
  annotations:
    reloader.stakater.com/auto: "true"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: haproxy
  template:
    metadata:
      labels:
        app: haproxy
      annotations:
        reloader.stakater.com/match: "true"
        #kubernetes.io/ingress-bandwidth: "5M"  # limit ingress 5 Mbps , หน่วย: M = Mbps, K = Kbps
        #kubernetes.io/egress-bandwidth: "5M"   # limit egress 5 Mbps , หน่วย: M = Mbps, K = Kbps
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - haproxy
                topologyKey: "kubernetes.io/hostname"
      containers:
        - name: haproxy
          image: haproxy:2.8
          args:
            - "-f"
            - "/usr/local/etc/haproxy/haproxy.cfg"
          ports:
            - name: http
              containerPort: 80
            - name: stats
              containerPort: 8404
```


## Create HAproxy configmap

```bash
vi haproxy-configmap.yaml
```

add yaml 

```yaml
####### HAproxy ConfigMap ##########
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: haproxy-config
  namespace: [your-name]-configmap
data:
  haproxy.cfg: |
    global
      log stdout format raw local0
      maxconn 5000

    defaults
      log global
      #mode http
      mode tcp
      #option httplog
      option tcplog
      option dontlognull
      timeout connect 5s
      timeout client  60s
      timeout server  60s
      timeout http-request 10s
      retries 3

    # ============================
    # Prometheus Metrics Endpoint
    # ============================
    frontend prometheus
      bind *:9101
      mode http
      http-request use-service prometheus-exporter

    # ============================
    #  Frontend (receive traffic)
    # # ============================
    frontend lb_frontend
      bind *:80
      mode tcp
    # ACL แยก health check
      acl is_probe hdr(User-Agent) -i kube-probe
      acl is_fw_health path_beg /health
      acl is_metrics path_beg /metrics
      use_backend lb_backend_health if is_probe or is_fw_health or is_metrics      
      default_backend lb_backend

    # ============================
    #  Backend (ceph rgw gateway replicas)
    # ============================
    backend lb_backend
      mode tcp
      balance roundrobin

      # ---- Active Health Check ----
      #option httpchk GET /api/health
      #http-check expect status 200
      #mode tcp

      # Backend nodes
      server ceph-rgw-01 172.16.50.241:8080 check
      server ceph-rgw-02 172.16.50.242:8080 check
      server ceph-rgw-03 172.16.50.243:8080 check
      #server grafana2 grafana-ha2:3000 check
      #server grafana3 grafana-ha3:3000 check

    # ============================
    #  Backend (ceph rgw gateway replicas heatch check)
    # ============================
    backend lb_backend_health
      mode tcp
      balance roundrobin

      # Health check tuning
      default-server inter 3s fall 3 rise 2

      server ceph-rgw-01 172.16.50.241:8080 check inter 3s fall 3 rise 2
      server ceph-rgw-02 172.16.50.242:8080 check inter 3s fall 3 rise 2
      server ceph-rgw-03 172.16.50.243:8080 check inter 3s fall 3 rise 2

    # ============================
    #  Stats Dashboard
    # ============================
    listen stats
      bind :8404
      mode http
      stats enable
      stats uri /stats
      stats refresh 5s
      stats auth admin:admin   # username : pass เปลี่ยนได้
```


Apply
```bash
kubectl apply -f haproxy-configmap.yaml
```

### Add Configmap to Deployment 

edit file my-test-volume-deployment.yaml

```bash
vi my-test-volume-deployment.yaml
```

Edit yaml ให้เพิ่ม volume ใหม่ที่เป็น configmap
```yaml
          volumeMounts:
            - name: haproxy-config
              mountPath: /usr/local/etc/haproxy/haproxy.cfg
              subPath: haproxy.cfg
         #readinessProbe:
         #   httpGet:
         #     path: /healthz
         #     port: 8404
         #   initialDelaySeconds: 2
         #   periodSeconds: 5
         # livenessProbe:
         #   httpGet:
         #     path: /healthz
         #     port: 8404
         #   initialDelaySeconds: 5
         #   periodSeconds: 10
      volumes:
        - name: haproxy-config
          configMap:
            name: s3-mek-01-config
```

Apply
```bash
kubectl apply -f my-test-volume-deployment.yaml
```

Check pod
```bash
kubectl get pod -n mypv
```

* ทดสอบ curl https://mytestvolume.user[x].olsdemo.com/api1  แล้วเกิดอะไรขึ้น ?


## Exercise 

* Create Nes Namespace : [yourname]-exercise-configmap
* Create Deployment,svc,httproute โดยใช้ images = registry-kubly.olsdemo.com/k8s101/nginx:1.29.2
* ให้สร้าง configmap ตาม yaml ด้านล่าง

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: index-html-configmap
  namespace: [yourname]-exercise-configmap
data:
  index.html: |
    <html>
    <h1>Welcome</h1>
    </br>
    <h1>Hi! This is a configmap Index file - Your name </h1>
    </html>
```
* map เข้าไปใน Deployment, Data Path ของ nginx คือ /usr/share/nginx/html
* HTTPRoute จะต้อง Secure
* แสดงผลผ่านหน้า Web Browser

## Navigation
* Previous: [Persistent Volume](/doc/09-persistent-volume.md)
* Next: [AutoScale](/doc/11-autoscale.md)
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