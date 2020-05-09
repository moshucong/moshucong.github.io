---
layout: post
title:  "容器内运行crontab的踩坑及解决"
date:   2018-04-17 16:12:13 +0800
categories: Docker
---

有些应用运行时需同时运行配套的crontab任务（如通过crontab定期更新数据）。这些应用搬到容器之后，crontab任务也就要运行在容器之中。



实践环境

- docker基础镜像：centos 6.9
- docker 1.10

## 1. 容器运行crontab思路


我们的思路很简单：在镜像内安装crontab，然后用实际要用的crontab任务配置文件替换镜像内root用户的crontab（/var/spool/cron/root）。

按照这样的思路，搞出了第一版的Dockerfile。

```
FROM centos:6.9
# Install cron
RUN yum install -y vixie-cron
... ...
ADD docker/crontab /var/spool/cron/root
ENTRYPOINT /proj/docker/start.sh
```

万万没想到，这样做出来镜像后，crontab任务并未能正常执行。



## 2. Trouble  Shooting

### 2.1 问题1：容器内rsyslog未启动，导致crontab写日志失败

#### 2.1.1 问题表现

crond日志默认保存于``/var/log/cron`` 。但容器启动后，发现``/var/log/cron``并不存在。

#### 2.1.2 原因 & 解决

crond调用``rsyslog``服务写日志。但在容器环境，crontab写日志所需的``rsyslog``服务默认并不启动。

为了让crond正常打log，需要在容器内启动rsyslog：

```
/etc/init.d/rsyslog start
```



### 2.2 问题2： PAM权限限制，导致crontab无法运行

#### 2.2.1 问题表现

重启rsyslog和crond后，容器内crontab任务仍然没有定时执行。

查看``/var/log/cron``  ，信息如下：

```
[root@0a95f33d7cb6 log]# cat cron
Dec 19 10:05:01 0a95f33d7cb6 crond[56]: (root) FAILED to open PAM security session (Cannot make/remove an entry for the specified session)
Dec 19 10:06:01 0a95f33d7cb6 crond[68]: (root) FAILED to open PAM security session (Cannot make/remove an entry for the specified session)
Dec 19 10:07:01 0a95f33d7cb6 crond[70]: (root) FAILED to open PAM security session (Cannot make/remove an entry for the specified session)
Dec 19 10:07:13 0a95f33d7cb6 crond[30]: (CRON) INFO (Shutting down)
Dec 19 10:07:13 0a95f33d7cb6 crond[97]: (CRON) STARTUP (1.4.4)
Dec 19 10:07:13 0a95f33d7cb6 crond[97]: (CRON) INFO (RANDOM_DELAY will be scaled with factor 3% if used.)
Dec 19 10:07:13 0a95f33d7cb6 crond[97]: (CRON) INFO (running with inotify support)
Dec 19 10:07:13 0a95f33d7cb6 crond[97]: (CRON) INFO (@reboot jobs will be run at computer's startup.)
```

#### 2.2.2 原因 & 解决
根据提示，PAM session认证未通过" (root) FAILED to open PAM security session"。查看crond ``/etc/pam.d/crond`` ：

```
# cat /etc/pam.d/crond
#
# The PAM configuration file for the cron daemon
#
#
# No PAM authentication called, auth modules not needed
account    required   pam_access.so
account    include    password-auth
session    required   pam_loginuid.so
session    include    password-auth
auth       include    password-auth
```

看情况，crontab应该通过了account认证，但挂在后面的session认证上面。尝试修改``/etc/pam.d/crond`` ，将其中的required改为sufficient：

```
# cat /etc/pam.d/crond
#
# The PAM configuration file for the cron daemon
#
#
# No PAM authentication called, auth modules not needed
account    sufficient    pam_access.so
account    include    password-auth
session    sufficient    pam_loginuid.so
session    include    password-auth
auth       include    password-auth
```

修改后重启crond，该错误消失。但crontab仍未能正常执行。



### 2.3 问题3： crontab任务文件访问权限不合要求，导致crontab任务加载失败

#### 2.3.1 问题表现

解决完上一个问题后，crontab仍然未正常执行，查看log如下。
```
[root@tesla-81250976-glmlq bin]# cat /var/log/cron 
Dec 19 10:12:40 tesla-81250976-glmlq crontab[30543]: (root) LIST (root)
Dec 19 10:13:03 tesla-81250976-glmlq crond[31]: (CRON) INFO (Shutting down)
Dec 19 10:13:03 tesla-81250976-glmlq crond[30624]: (CRON) STARTUP (1.4.4)
Dec 19 10:13:03 tesla-81250976-glmlq crond[30624]: (CRON) INFO (RANDOM_DELAY will be scaled with factor 0% if used.)
Dec 19 10:13:03 tesla-81250976-glmlq crond[30624]: (root) BAD FILE MODE (/var/spool/cron/root)
Dec 19 10:13:03 tesla-81250976-glmlq crond[30624]: (CRON) INFO (running with inotify support)
Dec 19 10:13:03 tesla-81250976-glmlq crond[30624]: (CRON) INFO (@reboot jobs will be run at computer's startup.)
Dec 19 10:13:07 tesla-81250976-glmlq crontab[30649]: (root) LIST (root)
Dec 19 10:13:50 tesla-81250976-glmlq crontab[30746]: (root) BEGIN EDIT (root)
Dec 19 10:14:23 tesla-81250976-glmlq crontab[30746]: (root) REPLACE (root)
Dec 19 10:14:23 tesla-81250976-glmlq crontab[30746]: (root) END EDIT (root)
Dec 19 10:15:01 tesla-81250976-glmlq CROND[30902]: (root) CMD (echo 'test' >> /home/test.log)
Dec 19 10:15:50 tesla-81250976-glmlq crontab[31013]: (root) BEGIN EDIT (root)
Dec 19 10:15:55 tesla-81250976-glmlq crontab[31013]: (root) REPLACE (root)
Dec 19 10:15:55 tesla-81250976-glmlq crontab[31013]: (root) END EDIT (root)
Dec 19 10:16:01 tesla-81250976-glmlq crond[30624]: (root) RELOAD (/var/spool/cron/root)
Dec 19 10:16:02 tesla-81250976-glmlq crontab[31038]: (root) LIST (root)
```

#### 2.3.2 原因 & 解决

日志提示：文件模式不对"BAD FILE MODE (/var/spool/cron/root)”，先看一下crontab文件模式：

```
[root@dbe293fb97f3 /]# ls /var/spool/cron/root -l
-rw-rw-r-- 1 root root 43 Dec 19 07:06 /var/spool/cron/root
```

经查阅资料得知，为了避免其他用户通过修改crontab获取root权限，除root用户外的其他用户均不能有写``/var/spool/cron/root``的权限，否则将带来安全问题。而我们容器内文件权限为664。

```
chmod 600 /var/spool/cron/root
```

重启crond，至此crontab正常执行。



## 3. 最终解决方案

修改基础镜像的Dockerfile ：

```
FROM centos:6.9

RUN yum install -y vixie-cron
... ...
# 覆盖自带的crond pam配置
ADD crond /etc/pam.d/crond   

ADD docker/crontab /var/spool/cron/root
# 修改crontab文件权限
RUN chmod 600 /var/spool/cron/root

ENTRYPOINT /proj/docker/start.sh
```

