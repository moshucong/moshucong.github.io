---
layout: post
title:  "Java容器的一些坑"
date:   2018-08-29 17:12:13 +0800
categories: Java
---

本文总结Java应用容器化的一些经验教训，其中大部分都与JVM内存配置有关。



### 1. JVM内存配置应依据docker内存限制自动设置

对于较低版本的JVM（1.8及以下版本）并未针对容器进行优化，在运维过程也有一些坑。具体可参考[为什么容器内的应用无法感知其内存限制](http://blog.allen-mo.com/2018/04/13/container_memory_limit/) 。 

最佳实践是：容器启动后，根据容器cgroup内存限制来动态配置JVM内存参数。

实践经验表明，对于一般java应用，按如下方案设置JVM内存参数，程序的运行较为稳定：

```
xmx = memory limit * 0.75
xms = xmx / 8 
xmn = xmx * 0.33 
xss = 1m 
```

参考资料：[Why does my Java process consume more memory than Xmx?](https://plumbr.io/blog/memory-leaks/why-does-my-java-process-consume-more-memory-than-xmx)



我们已经将这一计算过程写成一个脚本。容器中Java应用启动前执行这一脚本，即可自动计算出合理的JVM内存参数。使用示例如下：

```
. /docker-jvm-opts.sh
JAVA_OPTS="$JAVA_OPTS -Xms${JVM_XMS}m -Xmx${JVM_XMX}m -XX:PermSize=128M -XX:MaxPermSize=256m"

# 启动java应用，并传入JAVA_OPTS参数
java ${JAVA_OPTS} ************
```



### 2. 别忘记给tomcat配置内存参数

程序直接通过``java``命令启动时，我们通常都不会忘记加上JVM内存参数。但如果是部署到tomcat，有时会疏忽了JVM内存参数的配置。

``catalina.sh``启动时将读取``JAVA_OPTS``和``CATALINA_OPTS``环境变量：

```
#   CATALINA_OPTS   (Optional) Java runtime options used when the "start",
#                   "run" or "debug" command is executed.
#                   Include here and not in JAVA_OPTS all options, that should
#                   only be used by Tomcat itself, not by the stop process,
#                   the version command etc.
#                   Examples are heap size, GC logging, JMX ports etc.
#

#   JAVA_OPTS       (Optional) Java runtime options used when any command
#                   is executed.
#                   Include here and not in CATALINA_OPTS all options, that
#                   should be used by Tomcat and also by the stop process,
#                   the version command etc.
#                   Most options should go into CATALINA_OPTS.
```



因此，只需在启动脚本配置即可：

```
. /docker-jvm-opts.sh

JAVA_OPTS="$JAVA_OPTS -Xms${JVM_XMS}m -Xmx${JVM_XMX}m -XX:PermSize=128M -XX:MaxPermSize=256m"
export JAVA_OPTS=$JAVA_OPTS
```



### 3. 当JVM发生OutOfMemoryError，应让JVM自动退出

同事（@一超）反馈，某组件的JVM堆内存超出xmx限制，并抛``java.lang.OutOfMemoryError: Java heap space ``异常。堆内存爆了之后，JVM和java进程会继续运行，并不会crash，但实际已经无法正常提供服务。



解决方案：当JVM出现OutOfMemoryError，让JVM自行退出，而kubernetes将会立即重启容器。具体实施参考 [New JVM Options added: ExitOnOutOfMemoryError and CrashOnOutOfMemoryError](https://www.oracle.com/technetwork/java/javase/8u92-relnotes-2949471.html) ：

```
New JVM Options added: ExitOnOutOfMemoryError and CrashOnOutOfMemoryError
Two new JVM flags have been added:

ExitOnOutOfMemoryError - When you enable this option, the JVM exits on the first occurrence of an out-of-memory error. It can be used if you prefer restarting an instance of the JVM rather than handling out of memory errors.
```

修改在启动脚本中修改``JAVA_OPTS``，加上``-XX:+ExitOnOutOfMemoryError``：

```
JAVA_OPTS="$JAVA_OPTS -XX:+ExitOnOutOfMemoryError "
```



### 4. 健康检查尽量避免用jps命令

检查java进程是否存在是java应用健康检查的手段之一，java进程存在证明jvm还在运行。检查java进程可使用jvm提供的``jps``命令，也可以使用``ps``命令。

经测试，``jps``的执行效率远低于``ps``。



在某java应用容器内，执行``jps``的响应时间：

```
root@ym-boost-5447bb9494-6qlpv:~# time jps
37 Bootstrap
1119 Jps

real    0m0.265s
user    0m0.240s
sys     0m0.040s
```



在同一容器内，执行``ps``的响应时间：

```
root@ym-boost-5447bb9494-6qlpv:~# time ps -ef          
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 03:29 ?        00:00:00 /bin/sh -c bash /root/start.sh
root         6     1  0 03:29 ?        00:00:00 bash /root/start.sh
root        37     1 23 03:29 ?        00:04:30 /usr/bin/java -Djava.util.loggin
root        38     6  0 03:29 ?        00:00:00 /usr/sbin/sshd -D
root       580     0  0 03:38 ?        00:00:00 /bin/sh
root       586   580  0 03:38 ?        00:00:00 bash
root      1581   586  0 03:48 ?        00:00:00 ps -ef

real    0m0.008s
user    0m0.000s
sys     0m0.004s
```



在容器运行期间，liveness probe、readiness probe会被反复执行。考虑资源和效率，能用``ps``的情况下就尽量用``ps``，避免使用``jps``。

在我们的测试环境，当机器cpu load过高时，甚至出现了``jps``进程占用CPU达到100%，且长时间不结束的问题。



### 5. Scala等其他JVM语言的内存参数配置

除Java之外，任何采用JVM的应用都应该重视JVM内存参数的配置。

//TODO


