# Serverless 

Serverless คือ การรัน container หรือ Service แบบไม่ต้องดูแล infra มาก และ scale อัตโนมัติ ตามจำนวน request

## Install Knative Serving

* Install Custom Resource 

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.20.0/serving-crds.yaml
```

* Install Core Components of Knative Serving

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.20.0/serving-core.yaml
```

* Check Service 

```bash
kubectl get pod,svc -n knative-serving
```

