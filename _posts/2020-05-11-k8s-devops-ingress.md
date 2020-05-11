---
layout: post
title:  "成为Kubernetes运维开发（8）：Ingress"
date:   2020-05-11 08:14:13 +0800
categories: Kubernetes
---


在生产环境，Kubernetes上运行的应用有两种方式暴露到集群外：

* Loadbalancer。即通过LoadBalancer类型的K8s Service对外暴露服务。
* Ingress。HTTP七层代理。

Ingress是Kubernetes对7层反向代理的抽象。Kubernetes的Ingress有多种实现方式，最常见的是nginx。我们也可以通过与nginx类比，来理解Ingress。



### 1. Ingress基本操作

**1）创建Ingress**

创建文件``ingress.yml``，内容如下：

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: 【你的环境名字】
  namespace: 【你的环境名】
spec:
  rules:
  - host: 【你的环境名】.super-front.com
    http:
      paths:
      - backend:
          serviceName: super-front
          servicePort: 【你的super-front的Service端口】
        path: /
```

通过kubectl命令创建ingress：

````
kubectl apply -f ./ingress.yml
````



修改我们本地机器的``hosts``文件，添加如下的一行DNS配置。

```
172.30.10.41 【你的环境名】.super-front.com
```

例如我的windows系统，``hosts``文件在``C:\Windows\System32\drivers\etc``下。

打开浏览器，尝试访问``【你的环境名】.my-super-app.com``。可以访问到吗？



**2）查看Ingress**

``kubectl``命令查看ingress：

```
# kubectl get ing -n pg-allen
NAME       HOSTS                       ADDRESS   PORTS   AGE
pg-allen   pg-allen.my-super-app.com             80      6m48s
```



**3）配置TLS**

首先，需要一份TLS证书。下面以域名``pg-allen.super-front.com``为例。

创建秘钥：

```
openssl genrsa -out test-ingress.key 2048
```

创建证书请求（CSR）：

```
openssl req -new -key test-ingress.key -out test-ingress.csr \
        -subj "/CN=pg-allen.super-front.com"
```

创建证书：

```
openssl x509 -req -days 365 -in test-ingress.csr -signkey test-ingress.key \
        -out test-ingress.crt
```

至此，我们获得了秘钥``test-ingress.key``和证书``test-ingress.crt``。



然后，我们需创建Secret：

```
# kubectl -n pg-allen create secret tls pg-allen-super-front-com --cert ./test-ingress.crt --key ./test-ingress.key
secret/pg-allen-super-front-com created
```

查看配置的Secret，确认下：

```
# kubectl -n pg-allen get secret pg-allen-super-front-com -o yaml
apiVersion: v1
data:
  tls.crt: 【Base64编码的证书信息】
  tls.key: 【Base64编码的秘钥信息】
kind: Secret
metadata:
  creationTimestamp: "2020-04-11T03:11:21Z"
  name: pg-allen-super-front-com
  namespace: pg-allen
  resourceVersion: "357747872"
  selfLink: /api/v1/namespaces/pg-allen/secrets/pg-allen-super-front-com
  uid: 1e6909f5-7ba2-11ea-a111-0050568156a5
type: kubernetes.io/tls
```

注意：Secret的Type必须是``kubernetes.io/tls``，才能被Ingress所使用。



修改Ingress：

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: pg-allen
  namespace: pg-allen
spec:
  tls:
  - hosts:
    - pg-allen.super-front.com
    secretName: pg-allen-super-front-com
  rules:
  - host: pg-allen.super-front.com
    http:
      paths:
      - backend:
          serviceName: super-front
          servicePort: 18080
        path: /
```



尝试访问``https://pg-allen-super-front-com``，能否正常访问？（如果浏览器提示CA根不受信任，可忽略）



**4）nginx-ingress-controller原理初探**

为了理解ingress的原理，我们进入ingress-controller内部一探究竟。

在线下集群多个节点上，运行着``nginx-ingress-controller``。

```
# kubectl get po --all-namespaces -owide |grep ingress-controller 
ingress-nginx       nginx-ingress-controller-p9x9w                       1/1     Running                0          32d     172.30.10.41     172.30.10.41    <none>           <none>
ingress-nginx       nginx-ingress-controller-skf48                       1/1     Running                0          36d     172.30.10.69     172.30.10.69    <none>           <none>
... ...
```

我们进入其中一个ingress-controller pod内：

```
# kubectl exec -it nginx-ingress-controller-p9x9w  -n ingress-nginx bash
```

在ingress-controller pod内``/etc/nginx``目录下，我们看到``nginx.conf``文件。

在这个文件，可找到了``pg-allen.super-front.com``相关的配置：

```
        ## start server pg-allen.super-front.com
        server {
                server_name pg-allen.super-front.com ;

                listen 80;

                listen [::]:80;

                set $proxy_upstream_name "-";
                set $pass_access_scheme $scheme;
                set $pass_server_port $server_port;
                set $best_http_host $http_host;
                set $pass_port $pass_server_port;

                listen 443  ssl http2;

                listen [::]:443  ssl http2;

                # PEM sha: e2ec5127f5690d9b5be7d4c964745c25cac69e4b
                ssl_certificate                         /etc/ingress-controller/ssl/default-fake-certificate.pem;
                ssl_certificate_key                     /etc/ingress-controller/ssl/default-fake-certificate.pem;

                ssl_certificate_by_lua_block {
                        certificate.call()
                }

                location / {

                        set $namespace      "pg-allen";
                        set $ingress_name   "pg-allen";
                        set $service_name   "super-front";
                        set $service_port   "18080";
                        set $location_path  "/";

                        rewrite_by_lua_block {
                                lua_ingress.rewrite({
                                        force_ssl_redirect = true,
                                        use_port_in_redirects = false,
                                })
                                balancer.rewrite()
                                plugins.run()
                        }

                        header_filter_by_lua_block {

                                plugins.run()
                        }
                        body_filter_by_lua_block {

                        }

                        log_by_lua_block {

                                balancer.log()

                                monitor.call()

                                plugins.run()
                        }

                        if ($scheme = https) {
                                more_set_headers                        "Strict-Transport-Security: max-age=15724800; includeSubDomains";
                        }

                        port_in_redirect off;

                        set $balancer_ewma_score -1;
                        set $proxy_upstream_name    "pg-allen-super-front-18080";
                        set $proxy_host             $proxy_upstream_name;

                        set $proxy_alternative_upstream_name "";

                        client_max_body_size                    1m;

                        proxy_set_header Host                   $best_http_host;

                        # Pass the extracted client certificate to the backend

                        # Allow websocket connections
                        proxy_set_header                        Upgrade           $http_upgrade;

                        proxy_set_header                        Connection        $connection_upgrade;

                        proxy_set_header X-Request-ID           $req_id;
                        proxy_set_header X-Real-IP              $the_real_ip;

                        proxy_set_header X-Forwarded-For        $the_real_ip;

                        proxy_set_header X-Forwarded-Host       $best_http_host;
                        proxy_set_header X-Forwarded-Port       $pass_port;
                        proxy_set_header X-Forwarded-Proto      $pass_access_scheme;

                        proxy_set_header X-Original-URI         $request_uri;

                        proxy_set_header X-Scheme               $pass_access_scheme;

                        # Pass the original X-Forwarded-For
                        proxy_set_header X-Original-Forwarded-For $http_x_forwarded_for;

                        # mitigate HTTPoxy Vulnerability
                        # https://www.nginx.com/blog/mitigating-the-httpoxy-vulnerability-with-nginx/
                        proxy_set_header Proxy                  "";

                        # Custom headers to proxied server

                        proxy_connect_timeout                   5s;
                        proxy_send_timeout                      60s;
                        proxy_read_timeout                      60s;

                        proxy_buffering                         off;
                        proxy_buffer_size                       4k;
                        proxy_buffers                           4 4k;
                        proxy_request_buffering                 on;

                        proxy_http_version                      1.1;

                        proxy_cookie_domain                     off;
                        proxy_cookie_path                       off;

                        # In case of errors try the next upstream server before returning an error
                        proxy_next_upstream                     error timeout;
                        proxy_next_upstream_timeout             0;
                        proxy_next_upstream_tries               3;

                        proxy_pass http://upstream_balancer;

                        proxy_redirect                          off;

                }

        }
        ## end server pg-allen.super-front.com

```

ingress-controller将我们的ingress配置，转成了nginx（确切是openresty）的配置。

处理过程：外部访问——》nginx-ingress-controller（nginx转发）——》k8s Service——》pod（转发完成）




