## 简介

本章内容将把传统 php 项目转移到 k8s 集群中运行，将开发环境从 docker-compose 迁移到 k8s 。

### Docker Compose 下的 php

一个 docker-compose.yaml 文件搞定，主要是拆分为 nginx 和 fpm 容器，如果还用到其它服务，就需要额外引入，如 mysql ，redis，kafka，es 等。

docker-composer.yaml:

```yaml
version: '3'

networks:
  web-network:
    driver: bridge

services:
  fpm:
    build:
      context: ./fpm
    volumes:
      - "./nginx/apps:/var/www/html:cached"
      - "./fpm/php-fpm.d:/usr/local/etc/php-fpm.d:cached"
      - "./fpm/php.ini:/usr/local/etc/php/php.ini"
    networks:
      - web-network

  nginx:
    build:
      context: ./nginx
    ports:
      - '80:80'
    volumes:
      - ./nginx/apps:/var/www/html:cached
      - ./nginx/conf:/etc/nginx/conf.d:cached
      - ./nginx/log:/var/log/nginx
    networks:
      - web-network
```

就这样，一个最简单的 php 环境就搭好了，至于 Dockerfile ，直接根据官方镜像做自定义调整就好了，nginx 的话基本不需要调整，简单做下配置和代码的映射就好了；fpm 的话除了做配置和代码映射，还需要安装扩展和 composer 等基本工具。

fpm Dockerfile：

```dockerfile
FROM php:7.2-fpm

ADD install-php-extensions /usr/local/bin/

RUN apt-get update && apt-get install -y wget git vim ruby zlib1g-dev nodejs npm

RUN docker-php-ext-install pdo_mysql
RUN docker-php-ext-install bcmath
RUN docker-php-ext-install mysqli
RUN docker-php-ext-install sockets

RUN chmod uga+x /usr/local/bin/install-php-extensions && sync
RUN install-php-extensions redis
RUN install-php-extensions zip
RUN install-php-extensions pcov
RUN install-php-extensions pcntl
RUN install-php-extensions gd exif
RUN install-php-extensions xlswriter
RUN install-php-extensions swoole

COPY --from=composer:2.0.12 /usr/bin/composer /usr/local/bin/composer
```

fpm 官方镜像提供了 docker-php-ext-install 命令快速安装扩展，但是安装的扩展数量有限，推荐使用 install-php-extensions 脚本安装，几乎任何项目中用到的扩展都可以找得到。

nginx 怎样访问到 fpm 容器？因为使用的是同一个网络，所以直接使用服务名，nginx.conf:

```
upstream fpm {
    server fpm:9000;
}

server {
    listen       80;
    server_name  purelight.me;

    access_log  /var/log/nginx/purelight.me.access.log  main;
    error_log /var/log/nginx/purelight.me.error.log;
    root   /var/www/html/purelight/public;

    location / {
        index  index.html index.php;
        try_files $uri $uri/ /index.php$is_args$args;
    }
    
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }

    location ~ \.php$ {
        fastcgi_pass   fpm;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }
}
```

同样，对于 fpm 容器，就可以直接通过 mysql 服务名作为 host 去连接。

```docker-compose up -d ``` 就可以让 compose 完成镜像构建，容器启动，使用的 volume ，所以代码更改也可以实时看到效果，非常适合本地开发。

线上基本无法直接使用 docker compose ，因为它功能太少了，负载均衡，自动故障恢复，存活检测等都不太好完成，基本都是用 k8s ，所以考虑将本地环境升级到 k8s 。

### K8S 下的 php

K8S 里面是以 Pod 为最小单位，所以将 nginx 和 fpm 放进一个 pod ，做个简单的 k8s-demo 。

###### 创建命名空间

```sh
kubectl create namespace k8s-demo
```

###### 创建 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: k8s-demo
  name: k8s-com-deployment
  labels:
    app: k8s-com
    version: v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: k8s-com
      version: v1
  template:
    metadata:
      labels:
        app: k8s-com
        version: v1
    spec:
      containers:
        - name: nginx
          image: nginx:1.20.0
          ports:
            - containerPort: 80
          volumeMounts:
            - name: confd
              mountPath: /etc/nginx/conf.d
            - name: www
              mountPath: /var/www
            - name: logs
              mountPath: /var/log/nginx
        - name: fpm
          image: php:7.2-fpm
          ports:
            - containerPort: 9000
          volumeMounts:
            - name: www2
              mountPath: /var/www
      volumes:
        - name: confd
          hostPath:
            path: /Users/purelightme/Desktop/purelight-k8s/images/nginx/conf.d
        - name: www
          hostPath:
            path: /Users/purelightme/Desktop/purelight-k8s/images/nginx/www
        - name: www2
          hostPath:
            path: /Users/purelightme/Desktop/purelight-k8s/images/nginx/www
        - name: logs
          hostPath:
            path: /Users/purelightme/Desktop/purelight-k8s/images/nginx/logs
        - name: ini
          hostPath:
            path: /Users/purelightme/Desktop/purelight-k8s/images/fpm/php.ini
        - name: fpmd
          hostPath:
            path: /Users/purelightme/Desktop/purelight-k8s/images/fpm/php-fpm.d
```

因为是本地环境，所以选择 hostPath 挂载到 pod ，生产环境应该直接把配置和代码打包到镜像里面去。

###### 创建 Service

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: k8s-demo
  name: k8s-com-service
  labels:
    app: k8s-com
    version: v1
spec:
  type: NodePort
  selector:
    app: k8s-com
    version: v1
  ports:
   - port: 80
     targetPort: 80
```

这里 type 应该选择 ClusterIP (后面通过 ingress 暴露出去)，这里用 NodePort 的话就可以直接在本机访问了。

这里 selector 选择的是 pod 的标签，而不是 deployment 的。

###### 启动 nginx ingress controller

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.45.0/deploy/static/provider/cloud/deploy.yaml
```

###### 创建 Ingress

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: k8s-ing
  namespace: k8s-demo
spec:
  rules:
  - host: k8s.com
    http:
      paths:
      - path: /
        backend:
          serviceName: k8s-com-service
          servicePort: 80
```

本机配置 host 之后，就可以直接访问 http://k8s.com 了，ingress 捕捉到请求后将请求转发到 k8s-com-service service。同时也可以通过 ```kubectl get svc -n k8s-demo``` 找到 NodePort 的端口，访问 http://k8s.com:31245 也可以访问到服务，但是这种方式不会经过 ingress ，而是直接又 service 完成。

### 总结

不同的站点，我们应该用多个 pod 去实现，而不是把所有 site 都放进一个 pod 。Ingress 有很多配置项，支持7层代理，这样的话就可以实现动态负载均衡，其作为流量入口，可以提供前置和后置的功能，如认证授权，速率控制，日志搜集，统计等。

在传统的 web 开发里面，我们最后是要把代码上传到生产服务器上，在容器环境里面，我们就需要关注镜像了，可以说我们最后的产物就是打造一个又一个镜像，发布的过程就是更新 pod 镜像版本的过程。