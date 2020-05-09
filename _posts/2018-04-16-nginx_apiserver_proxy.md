---
layout: post
title:  "借助Nginx为k8s API-Server提供反向代理"
date:   2018-04-16 20:14:13 +0800
categories: Kubernetes
---


k8s采用多Master结构后，各个master上的API-SERVER监听各自的6443端口。因此需要引入反向代理，让k8s客户端通过这个反向代理与API-Server（相当于多个HTTPS后端服务）进行交互。在测试环境，我们采用Nginx作为k8s API-Server的反向代理。    

## 1. 原理介绍

常见的所谓“用Nginx实现HTTPS”实质上指的是利用Nginx的``SSL-Termination``，让网站对外提供HTTPS服务。即客户端浏览器与Nginx之间通过HTTPS通信，HTTPS数据在Nginx处解密成未加密的HTTP数据，后端服务器接收到的是HTTP请求。如下图所示。    

![SSL-Termination.png](https://raw.githubusercontent.com/moshucong/blog/master/images/SSL-Termination.png)

这种类反向代理的配置文件一般长这样：
```
server {

    listen   443;

    ssl    on;
    ssl_certificate    /etc/ssl/your_domain_name.pem;
    ssl_certificate_key    /etc/ssl/your_domain_name.key;

    server_name your.domain.com;
    location / {
        proxy_pass http://backend;
    }
}
```

但API-Server提供的是HTTPS服务，通过SSL Header获取API认证和授权所需的证书资料。如果采用Nginx ``SSL-Termination``，那么API-Server接收到的将是HTTP请求，SSL Header将会丢失，k8s API-Server认证和鉴权失败。所以，Nginx ``SSL-Termination``代理方式不适合k8s API-Server。



我们希望的负载均衡器仅仅充当请求转发的角色。

![SSL-Pass-thru.png](https://raw.githubusercontent.com/moshucong/blog/master/images/SSL-Pass-Through.png)        

Nginx的``SSL Pass-thru``代理方式正好能完美应对这种情况。    


## 2. 安装配置说明

下面介绍Nginx（with-stream）的两种安装配置方式：常规编译安装方式和docker部署方式。**由于docker部署方式简单快捷，因此推荐采用docker部署方式**，但常规编译安装方式也会进行介绍。

### 2.1 安装配置方式1：常规编译安装

**软件环境**：Ubuntu 64位 14.04 ， nginx-1.12.1（stable）

#### 2.1.1 编译安装Nginx（with-stream）

Nginx的预编译安装包并不包含``SSL Pass-thru``所需的stream模块，因此这里采用编译源码的方式安装Nginx。（The ngx_stream_core_module module is available since version 1.9.0. This module is not built by default, it should be enabled with the --with-stream configuration parameter.）    

设置工作空间为``/opt/working``：
```
export WORKSPACE=/opt/working
mkdir -p $WORKSPACE
```

安装nginx编译所需pcre库：
```
wget ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-8.38.tar.gz
tar -zxf pcre-8.38.tar.gz -C $WORKSPACE
cd $WORKSPACE/pcre-8.38
./configure
make
sudo make install
```

安装nginx编译所需zlib库：
```
wget http://zlib.net/zlib-1.2.11.tar.gz
tar -zxf zlib-1.2.11.tar.gz -C $WORKSPACE
cd $WORKSPACE/zlib-1.2.11
./configure
make
sudo make install
```

安装nginx编译所需OpenSSL库：
```
wget http://www.openssl.org/source/openssl-1.0.2f.tar.gz
tar -zxf openssl-1.0.2f.tar.gz -C $WORKSPACE
cd $WORKSPACE/openssl-1.0.2f
./Configure linux-x86_64 --prefix=/usr
sudo make install
```

编译安装nginx：
```
wget http://nginx.org/download/nginx-1.12.1.tar.gz
tar zxf nginx-1.12.1.tar.gz -C $WORKSPACE
cd $WORKSPACE/nginx-1.12.1
./configure \
--sbin-path=/usr/local/nginx/nginx \
--conf-path=/usr/local/nginx/nginx.conf \
--pid-path=/usr/local/nginx/nginx.pid \
--with-pcre=$WORKSPACE/pcre-8.38 \
--with-zlib=$WORKSPACE/zlib-1.2.11 \
--with-http_ssl_module \
--with-stream 
make
sudo make install
```

#### 2.1.2. 配置Nginx Stream代理

添加Nginx stream配置：
```
mkdir -p /usr/local/nginx/stream

cat << EOF > /usr/local/nginx/stream/master.conf
stream {
    upstream master {
        server 172.30.120.38:6444;
        server 172.30.120.39:6444;
        server 172.30.120.40:6444;
    }

    server {
        listen 6443;
        proxy_pass master;
    }
}
EOF
```
master的API-SERVER监听6444安全端口，反向代理监听6443端口。

修改nginx配置文件``/usr/local/nginx/nginx.conf``，将``master.conf`` include进去：

```
http {
    # ...
}
include /usr/local/nginx/stream/master.conf;
```

重新加载Nginx配置即可。



### 2.2 安装配置方式2：Docker方式

**软件环境**： docker

docker hub上找一个Nginx（with stream）的镜像，例如``tekn0ir/nginx-stream``，再写一个Dockerfile ：
```
FROM tekn0ir/nginx-stream
COPY master.conf /opt/nginx/stream.conf.d/master.conf
```
这里的``master.conf``就是上一节的那个配置文件。

编译镜像：
```
docker build -t="master-proxy:v0.01" .
```
启动反向代理：
```
docker run --restart=always --network="host" -d -p 0.0.0.0:6443:6443 --name master-proxy master-proxy:v0.01
```


Done！




## 3. 功能验证

向Nginx代理的HTTPS端口（6443）发送请求，查看请求能否被k8s API-Server正常处理：    
```
curl --key /opt/paas-deploy/cert/pem/admin-key.pem --cert /opt/paas-deploy/cert/pem/admin.pem --cacert /opt/paas-deploy/cert/pem/ca.pem https://172.30.120.38:6443/api/v1/nodes --verbose
```

若获得正常响应，则表明API-SERVER反向代理启动成功。


