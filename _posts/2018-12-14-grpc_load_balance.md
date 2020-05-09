---
layout: post
title:  "gRPC负载均衡解决方案"
date:   2018-12-14 12:12:13 +0800
categories: gRPC
---

本文总结gRPC负载均衡的特点，并介绍两种解决方案：nginx（with http2）和service mesh（istio）。

## 1. gRPC负载均衡有什么特殊？

首先，值得一读的是kubernetes官方关于gPRC负载均衡的一篇博文：[gRPC Load Balancing on Kubernetes without Tears](https://kubernetes.io/blog/2018/11/07/grpc-load-balancing-on-kubernetes-without-tears/)。这篇blog说明了kubernete service负载均衡机制对gRPC失效的原因：

```
However, gRPC also breaks the standard connection-level load balancing, including what’s provided by Kubernetes. 
This is because gRPC is built on HTTP/2, and HTTP/2 is designed to have a single long-lived TCP connection, across which all requests are multiplexed—meaning multiple requests can be active on the same connection at any point in time. 
Normally, this is great, as it reduces the overhead of connection management. 
However, it also means that (as you might imagine) connection-level balancing isn’t very useful. 
Once the connection is established, there’s no more balancing to be done. All requests will get pinned to a single destination pod ...
```

问题关键在于gRPC是基于HTTP/2长连接，gRPC客户端与服务端建立长连接后会保持并通过这个连接持续的发送请求。kubernetes service属于基于TCP连接级别的负载均衡（connection-level balancing），虽然支持gRPC通信，但其负载均衡的作用却会失效。



其次，可参考gRPC官方博客的这篇[gRPC Load Balancing](https://grpc.io/blog/loadbalancing) 。文末提出gRPC负载均衡的思路：

- 如果流量很高：客户端负载均衡（ZooKeeper/Etcd/Consul/Eureka等服务发现式的负载均衡）。
- 如果服务器需要对外提供统一的入口：代理式的负载均衡（haproxy 、nginx、Envoy）
- service mesh。




另外，k8s上的gRPC负载均衡可参考 [Grpc Loadbalancing Kubernetes Slides](https://github.com/jtattermusch/grpc-loadbalancing-kubernetes-examples/blob/master/grpc_loadbalancing_kubernetes_slides.pdf)  。




## 2. nginx方案

原理：gRPC客户端与nginx建立http/2连接后，nginx再与后端server建立多个http/2连接。

以下配置主要参考资料 [Introducing gRPC Support with NGINX 1.13.10](https://www.nginx.com/blog/nginx-1-13-10-grpc/) 。

### 2.1 安装配置nginx（with http_v2）

官方的标准安装包并不包含gRPC负载均衡所需的http_v2模块，因此需要编译安装。

安装nginx编译所依赖的pcre库：

```
export PCRE_WORKSPACE=/opt/working/pcre
mkdir -p $PCRE_WORKSPACE

cd $PCRE_WORKSPACE
wget ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-8.38.tar.gz
tar -zxf pcre-8.38.tar.gz -C $PCRE_WORKSPACE

cd $PCRE_WORKSPACE/pcre-8.38
./configure
make
sudo make install
ln -s /usr/local/lib/libpcre.so.1 /lib/libpcre.so.1
```


安装nginx编译所依赖的OpenSSL库：

```
export SSL_WORKSPACE=/opt/working/ssl
mkdir -p $SSL_WORKSPACE

cd $SSL_WORKSPACE
wget http://www.openssl.org/source/openssl-1.0.2f.tar.gz
tar -zxf openssl-1.0.2f.tar.gz -C $SSL_WORKSPACE

cd $SSL_WORKSPACE/openssl-1.0.2f
./Configure linux-x86_64 --prefix=/usr
sudo make install
```

配置编译nginx：

```
export NGINX_WORKSPACE=/opt/working/nginx
mkdir -p $NGINX_WORKSPACE

cd $NGINX_WORKSPACE
wget https://github.com/nginx/nginx/archive/release-1.15.7.tar.gz
tar -xvf ./release-1.15.7.tar.gz

cd $NGINX_WORKSPACE/nginx-release-1.15.7
auto/configure --with-http_ssl_module --with-http_v2_module 
make
sudo make install
```


### 2.2 配置nginx gRPC负载均衡

修改``/usr/local/nginx/conf/nginx.conf``，配置gRPC负载均衡：

```
#user nobody;
worker_processes  4;     # cpu core

error_log  logs/error.log;

pid        /run/nginx.pid;

events {
    worker_connections  1o24; # max connection =  worker_processes * worker_connections
#    multi_accept on; 
#    use epoll;  
}


http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent"';

    upstream grpc_servers {
        server 172.30.10.112:50051;
        server 172.30.10.113:50051;
    }
 
    server {
        listen 30051 http2;
 
        access_log logs/access.log main;
 
        location / {
            grpc_pass grpc://grpc_servers;
        }
    }
}
```

与http代理的主要区别是gRPC代理要用``grpc_pass``指令，协议要用``grpc://``，而 ``upstream``对gRPC同样适用。



## 3. service mesh（istio）方案

istio天生支持gRPC负载均衡。对于k8s集群中的应用，只需简单的配置即可启用该功能。



配置一个常规的k8s ``service``，需要注意的是要将port的名字改为``grpc``：

```
# kubectl get svc grpc-helloworld -n grpc-example -o yaml
apiVersion: v1
kind: Service
metadata:
  name: grpc-helloworld
  namespace: grpc-example
spec:
  externalIPs:
  - 172.30.10.123
  - 172.30.10.124
  ports:
  - name: grpc
    port: 50051
    protocol: TCP
    targetPort: 50051
  selector:
    app: grpc-helloworld
  sessionAffinity: None
  type: ClusterIP
```

配置一个istio ``gateway``，让其与istio ingressgateway的80端口关联：

```
# kubectl get gateway grpc-helloworld-gateway -n grpc-example -o yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: grpc-helloworld-gateway
  namespace: grpc-example
  uid: e5613eb1-fecb-11e8-854e-0050568156a5
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - '*'
    port:
      name: grpc
      number: 80
      protocol: HTTP
```

配置一个istio ``VirtualService``，将istio ``gateway``与k8s ``service``关联起来：

```
# kubectl get virtualservice grpc-helloworld -n grpc-example -o yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: grpc-helloworld
  namespace: grpc-example
spec:
  gateways:
  - grpc-helloworld-gateway
  hosts:
  - '*'
  http:
  - match:
    - uri:
        regex: .*
    route:
    - destination:
        host: grpc-helloworld
        port:
          number: 50051
```



只需上述配置，gRPC负载均衡就好了。
