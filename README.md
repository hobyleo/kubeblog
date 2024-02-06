# KubeBlog

一个博客系统，由 Spring Boot + MySQL + Semantic UI 组成。目的是将该博客运行在 Kubernetes 环境里，从而学习和实践 Kubernetes 的概念。

访问方式：

用户端：http://localhost:5000

管理员：http://localhost:5000/admin

账号/密码：admin/password

git 地址：https://gitee.com/hobyleo/kubeblog.git

## 环境准备
### 软硬件配置
| 软硬件 | 配置/版本 | 备注  |
|  ----  | ----  |----  |
| 虚拟机平台 | VMWare Workstation 16.x |注：不能用 VirtualBox，iptables 转发会有问题 |
| 操作系统 | CentOS 7.9 |http://mirrors.aliyun.com/centos/7/isos/x86_64/CentOS-7-x86_64-DVD-2009.iso |
| 虚拟机配置 | 2 核 4G |注：k8s 最低要求 cpu 要 2 核以上 |
| JDK | 1.8 | |
| Maven | 3.6.3 |                                                              |
| MySQL | 5.7.36 | root/password |



### 域名映射
|  ip  |  域名  | 访问路径  |
|  ----  | ----  |----  |
|  127.0.0.1  | art.local | http://art.local:8081 |
|  192.168.10.101 | master | |
|  192.168.10.102 | node1 | |
|  192.168.10.103 | node2 | |



### maven

- 执行

```
yum install -y maven
```

- 配置阿里云镜像

  vi /etc/maven/settings.xml，添加 mirrors

```
    <mirror>
      <id>alimaven</id>
      <name>alimaven</name>
      <mirrorOf>central</mirrorOf>
      <url>https://maven.aliyun.com/repository/public/</url>
    </mirror>

    <mirror>
      <id>huaweicloud</id>
      <name>huaweicloud</name>
      <mirrorOf>central</mirrorOf>
      <url>https://mirrors.huaweicloud.com/repository/maven/</url>
    </mirror>
```



### my.cnf

```
[mysqld]
default-time-zone='+08:00'
bind-address=0.0.0.0
port=3306
character-set-server=utf8mb4
collation-server=utf8mb4_general_ci
max_connections=500
max_allowed_packet=512M

[mysql]
default-character-set=utf8mb4

[client]
default-character-set=utf8mb4
```



## NFS

- 在 master 和所有 worker node 安装 nfs 服务：

```
yum install -y nfs-utils rpcbind
```

- 创建 nfs 目录

```
mkdir /nfsdata
```

- 配置 nfs 目录


```
echo /nfsdata   *(rw,sync,no_root_squash) >> /etc/exports
```

- 激活配置

```
exportfs -r
```

- 在 master 自启动 nfs 和 rpcbind

```
systemctl enable --now rpcbind
systemctl enable --now nfs
```



## 私有镜像中心

- 在 master 上执行

```
mkdir ~/JFROG_HOME/
```

- 添加环境变量

  vi ~/.bash_profile

```
export JFROG_HOME=~/JFROG_HOME
```

​	source ~/.bash_profile

- 创建目录

```
mkdir -p $JFROG_HOME/artifactory/var/etc/
cd $JFROG_HOME/artifactory/var/etc/
touch ./system.yaml
chmod -R 777 $JFROG_HOME/artifactory/var
```

- 下载镜像

```
docker pull releases-docker.jfrog.io/jfrog/artifactory-jcr:7.33.9
```

- 启动 JCR

```
docker run --name artifactory-jcr \
-p 8081:8081 -p 8082:8082 \
-v $JFROG_HOME/artifactory/var/:/var/opt/jfrog/artifactory \
-d --restart=always releases-docker.jfrog.io/jfrog/artifactory-jcr:7.33.9
```

- 查看启动日志

```
docker logs -f artifactory-jcr
```

- 打开 Web UI

  http://192.168.10.101:8082/ui/login/

  账号/密码：admin/password

- 解决 `No signed EULA found. To activate, proceed to sign the EULA.`，在 master 执行：

```
curl -XPOST -vu admin:password http://localhost:8081/artifactory/ui/jcr/eula/accept
```

​	看到 200 说明成功：

```
[root@master etc]# curl -XPOST -vu admin:password http://localhost:8081/artifactory/ui/jcr/eula/accept
* About to connect() to localhost port 8081 (#0)
*   Trying ::1...
* Connected to localhost (::1) port 8081 (#0)
* Server auth using Basic with user 'admin'
> POST /artifactory/ui/jcr/eula/accept HTTP/1.1
> Authorization: Basic YWRtaW46cGFzc3dvcmQ=
> User-Agent: curl/7.29.0
> Host: localhost:8081
> Accept: */*
> 
< HTTP/1.1 200 
< X-JFrog-Version: Artifactory/7.33.9 73309900
< X-Artifactory-Id: 90093da3beab108c:f60fd29:18d7429e4bd:-8000
< X-Artifactory-Node-Id: 7a41af5083c5
< SessionValid: false
< Content-Length: 0
< Date: Sun, 04 Feb 2024 12:52:47 GMT
< 
* Connection #0 to host localhost left intact
[root@master etc]#
```

- 创建五个仓库
  - docker-local
  - docker-test
  - docker-release
  - docker-remote
  - docker

- 推送镜像到仓库

```
docker login art.local:8081
```

```
docker tag kubeblog:latest art.local:8081/docker-local/kubeblog:latest
```

```
docker push art.local:8081/docker-local/kubeblog:latest
```

- 创建镜像仓库密钥

```
kubectl create secret docker-registry regcred-local --docker-server=192.168.10.101:8081 --docker-username=admin --docker-password=password
```

- 创建 mysql 密钥

```
kubectl create secret generic mysql-password-test --from-literal=MYSQL_PASSWORD_TEST=password
```



## Helm

- 安装

```
wget https://get.helm.sh/helm-v3.4.2-linux-amd64.tar.gz -O helm-v3.4.2-linux-amd64.tar.gz
tar -zxvf  helm-v3.4.2-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
```

- [官方文档](https://helm.sh/zh/docs/intro/quickstart/)



## K8s 升级

### 升级原则

- 不能跨次版本升级，比如 1.19 不能跨过 1.20 升级到 1.21，必须先升级到 1.20.X 版本
- 升级时不要进行应用的操作
- 确保 swap 的关闭
- 为虚拟机镜像创建 snapshot，以便出错后可以回滚

### 升级 master

1. 查看新版本

```
yum list --showduplicates kubeadm --disableexcludes=kubernetes
```

2. 升级 kubeadm

```
yum install -y kubeadm-1.20.15-0 --disableexcludes=kubernetes
```

3. 查看 kubeadm 版本

```
[root@master ~]# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.15", GitCommit:"8f1e5bf0b9729a899b8df86249b56e2c74aebc55", GitTreeState:"clean", BuildDate:"2022-01-19T17:26:37Z", GoVersion:"go1.15.15", Compiler:"gc", Platform:"linux/amd64"}
```

4. 执行升级检测

```
kubeadm upgrade plan
```

5. 执行升级

```
kubeadm upgrade apply v1.20.15
```

6. 升级 kubectl 和 kubelet

```
yum install -y kubectl-1.20.15-0 kubelet-1.20.15-0 --disableexcludes=kubernetes
```

7. 重启 kubelet

```
systemctl daemon-reload && systemctl restart kubelet
```

8. 查看 kubectl 版本已经更新

```
[root@master ~]# kubectl version
Client Version: version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.15", GitCommit:"8f1e5bf0b9729a899b8df86249b56e2c74aebc55", GitTreeState:"clean", BuildDate:"2022-01-19T17:27:39Z", GoVersion:"go1.15.15", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.3", GitCommit:"1e11e4a2108024935ecfcb2912226cedeafd99df", GitTreeState:"clean", BuildDate:"2020-10-14T12:41:49Z", GoVersion:"go1.15.2", Compiler:"gc", Platform:"linux/amd64"}
```

### 升级 worker

1. 升级 kubeadm

```
yum install -y kubeadm-1.20.15-0 --disableexcludes=kubernetes
```

2. 升级 node

```
kubeadm upgrade node
```

3. 升级 kubectl 和 kubelet

```
yum install -y kubectl-1.20.15-0 kubelet-1.20.15-0 --disableexcludes=kubernetes
```

4. 重启

```
systemctl daemon-reload && systemctl restart kubelet
```

5. 验证

> 等待 5 秒执行

```
[root@node1 ~]# kubectl get node
NAME     STATUS   ROLES    AGE   VERSION
master   Ready    master   13h   v1.20.15
node1    Ready    <none>   13h   v1.20.15
```

