---
layout: post
title:  "Kubernetes WebSocket探究&应用"
date:   2018-04-17 17:12:13 +0800
categories: Kubernetes
---

WebSocket协议与HTTP的主要区别：HTTP是无状态协议，由客户端发起请求，客户端与服务器“一问一答”，因此服务器端无法主动向客户端发送信息。而WebSocket是基于TCP长连接的协议，客户端与服务器建立连接后，服务器端随时能向客户端发送信息。

WebSocket协议的主要价值在于其与HTTP的差异（服务器端与客户端能够保持实时的双向通信），使其在某些应用情景下比HTTP更能满足技术需求。



## 1. Kubernetes WebSocket研究

Kubernetes提供了两个WebSocket端点：

- exec端点：可远程在容器中执行命令，并实时获取命令执行过程中的输出。


- attach端点：实时查看容器的log输出。

### 1.1 exec端点

exec端点的作用类似于``kubectl exec``命令。通过WebSocket连接该端点后，可在容器内打开tty并执行命令。该端点URI如下：
```
/api/v1/namespaces/{namespace}/pods/{name}/exec?stdout={}&stdin={}&stderr={}&tty={}&container={}&command={}
```
完整URI示例如下：
```
ws://172.30.80.95:8080/api/v1/namespaces/default/pods/ubuntu-1/exec?stdout=1&stdin=1&stderr=1&tty=1&command=/bin/sh
```

端点中的参数说明如下（来自k8s swagger）：

| 参数        | 说明                                       |
| --------- | ---------------------------------------- |
| stdin     | Redirect the standard input stream of the pod for this call. Defaults to false. |
| stdout    | Redirect the standard output stream of the pod for this call. Defaults to true. |
| stderr    | Redirect the standard error stream of the pod for this call. Defaults to true. |
| tty       | TTY if true indicates that a tty will be allocated for the exec call. Defaults to false. |
| container | Container in which to execute the command. Defaults to only container if there is only one container in the pod. |
| command   | Command is the remote command to execute. argv array. Not executed within a shell. |
| namespace | object name and auth scope, such as for teams and projects |
| name      | name of the Pod                          |



### 1.2 attach端点

attach端点的作用与``kubectl attach``命令类似。通过WebSocket连接该端点后，可实时查看容器输出到标准输出（stdout、stderr）的log信息。该端点URI如下：

```
/api/v1/namespaces/{namespace}/pods/{name}/attach?stdout={}&stdin={}&stderr={}&tty={}&container={}&command={}
```

该端点完整的URI示例如下：

```
ws://172.30.80.95:8080/api/v1/namespaces/default/pods/ubuntu-1/attach?stdout=1&stdin=1&stderr=1&tty=1&command=/bin/sh
```

端点参数说明如下（来自k8s swagger）：


| 参数        | 说明                                       |
| --------- | ---------------------------------------- |
| stdin     | Stdin if true, redirects the standard input stream of the pod for this call. Defaults to false. |
| stdout    | Stdout if true indicates that stdout is to be redirected for the attach call. Defaults to true. |
| stderr    | Stderr if true indicates that stderr is to be redirected for the attach call. Defaults to true. |
| tty       | TTY if true indicates that a tty will be allocated for the attach call. This is passed through the container runtime so the tty is allocated on the worker node by the container runtime. Defaults to false. |
| container | The container in which to execute the command. Defaults to only container if there is only one container in the pod. |
| namespace | object name and auth scope, such as for teams and projects |
| name      | name of the Pod                          |



### 1.3. 利用wscat测试Kubernetes WebSocket端点

对于WebSocket测试，Postman、JMeter等HTTP测试工具已经不再适用。这里我们通过wscat工具尝试连接Kubernetes WebSocket端点，以便获得一个初步的认识。

安装``wscat``：

```
apt-get update
apt-get install -y npm
ln -s /usr/bin/nodejs /usr/bin/node
npm install -g n
n stable
npm install -g wscat
```

首先，尝试通过k8s非安全端口（8080）连接容器：

```
# wscat -c "ws://172.30.80.95:8080/api/v1/namespaces/default/pods/ubuntu-1/exec?container=ubuntu-1&stdin=1&stdout=1&stderr=1&tty=1&command=ls"
connected (press CTRL+C to quit)
  <  
  <  
  < bin   dev  home  lib64	mnt  proc  run	 srv  tmp  var
boot  etc  lib	 media	opt  root  sbin  sys  usr

  disconnected
```

成功在容器内执行了``ls``命令，证明WebSocket连接成功。

然后，尝试通过k8s安全端口（6443）连接容器：

```
# wscat -c "wss://172.30.80.95:6443/api/v1/namespaces/default/pods/ubuntu-1/exec?container=ubuntu-1&stdin=1&stdout=1&stderr=1&tty=1&command=ls" --key ./admin-key.pem --cert ./admin.pem --ca ./ca.pem
connected (press CTRL+C to quit)
  <  
  <  
  < bin   dev  home  lib64	mnt  proc  run	 srv  tmp  var
boot  etc  lib	 media	opt  root  sbin  sys  usr

  disconnected
```

注意，由于连接的是安全端口，因此在header中携带了k8s集群的证书。



再尝试连接attach接口（非安全端口）：

```
# wscat -c "ws://172.30.80.95:8080/api/v1/namespaces/kube-system/pods/etcd-allen-laptop/attach?container=etcd&stdin=1&stdout=1&stderr=1&tty=1"
connected (press CTRL+C to quit)
  <  
  < 2017-10-20 07:01:53.457541 W | etcdserver: apply entries took too long [101.264318ms for 1 entries]
2017-10-20 07:01:53.457598 W | etcdserver: avoid queries with large range/delete range!

  < 2017-10-20 07:01:55.539483 W | etcdserver: apply entries took too long [44.611169ms for 1 entries]
2017-10-20 07:01:55.539555 W | etcdserver: avoid queries with large range/delete range!

> 
```

可看到，连接``attach``接口后，将持续输出容器的log。



最后，尝试通过k8s安全端口连接attach接口：

````
# wscat -c "wss://172.30.80.95:6443/api/v1/namespaces/kube-system/pods/etcd-allen-laptop/attach?container=etcd&stdin=1&stdout=1&stderr=1&tty=1" --key ./admin-key.pem --cert ./admin.pem --ca ./ca.pem


connected (press CTRL+C to quit)
  <  
  < 2017-10-20 07:05:31.500131 W | etcdserver: apply entries took too long [31.227698ms for 1 entries]
2017-10-20 07:05:31.500173 W | etcdserver: avoid queries with large range/delete range!

  < 2017-10-20 07:05:34.548632 W | etcdserver: apply entries took too long [118.130233ms for 1 entries]
2017-10-20 07:05:34.548680 W | etcdserver: avoid queries with large range/delete range!

> 
````



### 1.4 Kubernetes WebSocket子协议

Kubernetes WebSocket接口遵循WebSocket协议，并在此基础上设计了子协议。该子协议协议并没有官方文档，但我们可以从k8s的代码（ https://github.com/kubernetes/apiserver/blob/master/pkg/util/wsstream/conn.go ）中找到相关说明：

```
// The Websocket subprotocol "channel.k8s.io" prepends each binary message with a byte indicating
// the channel number (zero indexed) the message was sent on. Messages in both directions should
// prefix their messages with this channel byte. When used for remote execution, the channel numbers
// are by convention defined to match the POSIX file-descriptors assigned to STDIN, STDOUT, and STDERR
// (0, 1, and 2). No other conversion is performed on the raw subprotocol - writes are sent as they
// are received by the server.
//
// Example client session:
//
//    CONNECT http://server.com with subprotocol "channel.k8s.io"
//    WRITE []byte{0, 102, 111, 111, 10} # send "foo\n" on channel 0 (STDIN)
//    READ  []byte{1, 10}                # receive "\n" on channel 1 (STDOUT)
//    CLOSE
//
const ChannelWebSocketProtocol = "channel.k8s.io"

// The Websocket subprotocol "base64.channel.k8s.io" base64 encodes each message with a character
// indicating the channel number (zero indexed) the message was sent on. Messages in both directions
// should prefix their messages with this channel char. When used for remote execution, the channel
// numbers are by convention defined to match the POSIX file-descriptors assigned to STDIN, STDOUT,
// and STDERR ('0', '1', and '2'). The data received on the server is base64 decoded (and must be
// be valid) and data written by the server to the client is base64 encoded.
//
// Example client session:
//
//    CONNECT http://server.com with subprotocol "base64.channel.k8s.io"
//    WRITE []byte{48, 90, 109, 57, 118, 67, 103, 111, 61} # send "foo\n" (base64: "Zm9vCgo=") on channel '0' (STDIN)
//    READ  []byte{49, 67, 103, 61, 61} # receive "\n" (base64: "Cg==") on channel '1' (STDOUT)
//    CLOSE
//
const Base64ChannelWebSocketProtocol = "base64.channel.k8s.io"
```



从这段注释可知，**Kubernetes设计有两种WebSocket子协议，分别叫"channel.k8s.io"和"base64.channel.k8s.io"**。

* "channel.k8s.io"协议在发送/接收消息时，以消息的第一个byte来表示消息的类型，0表示输入STDIN，1表示标准输出STDOUT，2表示标准错误输出STDERR。
* "base64.channel.k8s.io"协议与“channel.k8s.io”类似，区别在于第一个byte用字符表示消息类型，字符'0'表示输入STDIN，字符'1'表示标准输出STDOUT，字符'2'表示标准错误输出STDERR；消息内容经过Base64编码。



在编写WebSocket客户端与Kubernetes通信时，必须选择并遵守其中一种协议。



## 2. 客户端代码示例

以JavaScript为例说明与Kubernetes WebSocket的交互。

### 2.1 客户端初始化

JavaScript建立WebSocket客户端：

```
var wsUrl = "ws://172.30.80.95:8080/api/v1/namespaces/default/pods/ubuntu-1/exec?container=ubuntu-1&stdin=1&stdout=1&stderr=1&tty=1&command=/bin/sh" ;
var ws = new WebSocket(url,"base64.channel.k8s.io") ;
```

其中，协议实际上将作为header "Sec-WebSocket-Protocol : [base64.channel.k8s.io](http://base64.channel.k8s.io/)"传输。



### 2.2 发送信息

JavaScript WebSocket发送信息：
```
ws.send("0" + btoa( str ) );
```

### 2.3 接收并处理信息

JavaScript WebSocket接收信息：

```
        ws.onmessage = function(event) {
            data = event.data.slice(1);
            switch(event.data[0]) {
            case '1':
            case '2':
            case '3':
                console.log( 'received:' + atob(data))
                break;
            }
        };
```

**关键在于，建立连接、发送信息、接收处理信息都要遵守Kubernetes的WebSocket子协议。**
