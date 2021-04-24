## Docker 容器化基础



## K8S 容器编排



## 实践出真知

###### 安装 docker for mac 并启用 k8s

https://github.com/maguowei/k8s-docker-desktop-for-mac

###### 构建 ds_ngx 镜像，push 到 docker hub

```dockerfile
FROM nginx:1.19.8

COPY html /usr/share/nginx/html
```

```
docker build . -t purelightme/ds_ngx:v1 
```

```
docker push purelightme/ds_ngx:v1
```

https://hub.docker.com/

###### 创建 Deployment

ds_ngx_deployment.yml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
  labels:
    app: ngx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ngx
  template:
    metadata:
      labels:
        app: ngx
    spec:
      containers:
        - name: ngx
          image: purelightme/ds_ngx:v1
          ports:
            - containerPort: 80
```

```
kubectl apply -f ds_ngx_deployment.yml
kubectl get deployment
```

###### 创建 Service

ds_ngx_service.yml

```
apiVersion: v1
kind: Service
metadata:
  name: ngx-service
spec:
  selector:
    app: ngx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort

```

```
kubectl apply -f ds_ngx_service.yml
kubectl get svc
```

![ds_ngx_svc](/Users/purelightme/Desktop/k8s/share/ds_ngx_svc.png)

http://localhost:30351/ds.html

## 推向生产环境

https://cloud.tencent.com/