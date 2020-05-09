## 一、脚本由来

在参考了<https://github.com/opsnull/follow-me-install-kubernetes-cluster>手动部署完k8s-1.16.6高可用集群之后，就想着是否要自己试着写一下部署脚本，毕竟安装的步骤那么多，如果能有一个自动化的部署脚本，那以后部署的时候真的是轻松和快速多了。

在犹豫之际，在"二丫讲梵"的博客中就看到了这篇[k8s-1.10.4一键部署脚本](http://www.eryajf.net/2231.html)的文章。细品之后，就更加坚定了自己写k8s-1.16.6一键部署脚本的信念。

在此，再次感谢两位大佬所做的分享，我的一键部署脚本，主要是参考了这两位的文章。



## 二、环境

| 服务器IP     | 系统      | 主机名 | 组件                                                         |
| ------------ | --------- | ------ | ------------------------------------------------------------ |
| 192.168.0.71 | CentOS7.6 | k8s-01 | Kubernetes 1.16.6,Docker 18.09.6,Etcd 3.3.20,Flanneld 0.11.0,kube-apiserver,kube-controller-manager,kube-scheduler,kubelet,kube-proxy,nginx-1.15.3 |
| 192.168.0.72 | CentOS7.6 | k8s-02 | 同上                                                         |
| 192.168.0.73 | CentOS7.6 | k8s-03 | 同上                                                         |



## 三、准备工作

首先把整个部署文件上传到服务器上，进行解压，然后做以下的准备工作。

其中的脚本代码，我已经上传到Github中 ：

#### 1、修改以下内容

```shell
config/environment.sh #修改ip为自己服务器的ip

config/Kcsh/hosts #修改ip为自己服务器的ip

config/Ketcd/etcd-csr.json #修改ip为自己服务器的ip

config/Kapi/kubernetes-csr.json #修改ip为自己服务器的ip

config/Kmanage/kube-controller-manager-csr.json #修改ip为自己服务器的ip

config/Kscheduler/kube-scheduler-csr.json #修改ip为自己服务器的ip

config/Kha/kube-nginx.conf #修改ip为自己服务器的ip
```

#### 2、基础配置

这些操作**均在 k8s-01 节点**上执行即可。

```shell
ssh-keygen
ssh-copy-id 192.168.0.71
ssh-copy-id 192.168.0.72
ssh-copy-id 192.168.0.73

scp config/Kcsh/hosts root@192.168.0.71:/etc/hosts
scp config/Kcsh/hosts root@192.168.0.72:/etc/hosts
scp config/Kcsh/hosts root@192.168.0.73:/etc/hosts

ssh root@k8s-01 "hostnamectl set-hostname k8s-01"
ssh root@k8s-02 "hostnamectl set-hostname k8s-02"
ssh root@k8s-03 "hostnamectl set-hostname k8s-03"
#然后退出重新登录，可以看到服务器的主机名已经改好
```

#### 3、升级内核

CentOS7.x系统自带的额3.10.x内核存在一些Bug，导致运行的Docker、Kubernetes不稳定，例如：

1）高版本的docker(1.13以后)启用了 3.10 kernel 实验支持的 kernel memory account 功能(无法关闭)，当节点压力大如频繁启动和停止容器时会导致 cgroup memory leak；

2）网络设备引用计数泄漏，会导致类似于报错："kernel:unregister_netdevice: waiting for eth0 to become free. Usage count = 1";

所以方便的话，最好还是升级下内核。

这些操作需要在**所有主机**上都执行。

```shell
yum -y install https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
# 安装完成后检查 /boot/grub2/grub.cfg 中对应内核 menuentry 中是否包含 initrd16 配置，如果没有，再安装一次！
yum --enablerepo=elrepo-kernel install -y kernel-lt
# 设置开机从新内核启动
grub2-set-default 0

# 重启主机
sync
reboot
```



## 四、正式部署

正式部署非常的简单地，直接执行`install.sh`脚本就行。

不过在正式部署之前，需要保证已经完成上面的**准备工作**，不然部署的时候会报错哦！



## 五、简单验证

部署完成之后，可使用下面的步骤对集群的功能和可用性做一下验证。

#### 1、检查服务的状态

```shell
#!/bin/bash

source /opt/k8s/bin/environment.sh

##set color##
echoRed() { echo $'\e[0;31m'"$1"$'\e[0m'; }
echoGreen() { echo $'\e[0;32m'"$1"$'\e[0m'; }
echoYellow() { echo $'\e[0;33m'"$1"$'\e[0m'; }
##set color##

for node_ip in ${NODE_IPS[@]}
do
    echoGreen ">>> ${node_ip}"
    ssh root@${node_ip} "systemctl status etcd|grep Active"
    ssh root@${node_ip} "systemctl status flanneld|grep Active"
    ssh root@${node_ip} "systemctl status kube-apiserver|grep Active"
    ssh root@${node_ip} "systemctl status kube-controller-manager|grep Active"
    ssh root@${node_ip} "systemctl status kube-scheduler|grep Active"
    ssh root@${node_ip} "systemctl status kube-nginx|grep Active"
    ssh root@${node_ip} "systemctl status docker|grep Active"
    ssh root@${node_ip} "systemctl status kubelet|grep Active"
    ssh root@${node_ip} "systemctl status kube-proxy|grep Active"
done
```

#### 2、检查相关服务的可用性

##### 1）验证etcd服务的可用性

```shell
cat > deploy.sh << "EOF"
#!/bin/bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh 
for node_ip in ${NODE_IPS[@]}
do
    echo ">>> ${node_ip}"
    ETCDCTL_API=3 /opt/k8s/bin/etcdctl \
    --endpoints=https://${node_ip}:2379 \
    --cacert=/etc/kubernetes/cert/ca.pem \
    --cert=/etc/etcd/cert/etcd.pem \
    --key=/etc/etcd/cert/etcd-key.pem endpoint health
done
EOF
```

##### 2）验证flannel网络

查看已经分配的 Pod 子网段列表：

```shell
$ source /opt/k8s/bin/environment.sh

$ etcdctl \
  --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/cert/ca.pem \
  --cert-file=/etc/flanneld/cert/flanneld.pem \
  --key-file=/etc/flanneld/cert/flanneld-key.pem \
  ls ${FLANNEL_ETCD_PREFIX}/subnets
```

输出（结果视部署情况而定）：

```shell
/subnets/172.30.128.0-21
/subnets/172.30.88.0-21
/subnets/172.30.104.0-21
```

验证各节点能通过 Pod 网段互通

`注意：其中的IP段换成上面输出的IP段`

```shell
cat > deploy.sh << "EOF"
#!/bin/bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh 
for node_ip in ${NODE_IPS[@]}
do
    echo ">>> ${node_ip}"
    ssh ${node_ip} "ping -c 1 172.30.128.0"
    ssh ${node_ip} "ping -c 1 172.30.88.0"
    ssh ${node_ip} "ping -c 1 172.30.104.0"
done
EOF
```

##### 3）高可用性试验

查看当前的leader：

```shell
$ kubectl get endpoints kube-controller-manager --namespace=kube-system  -o yaml
apiVersion: v1
kind: Endpoints
metadata:
  annotations:
    control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"k8s-01_130e5bf0-8d2a-42d5-86a4-9ef84c16e641","leaseDurationSeconds":15,"acquireTime":"2020-04-24T05:31:47Z","renewTime":"2020-04-24T05:42:44Z","leaderTransitions":0}'
  creationTimestamp: "2020-04-24T05:31:47Z"
  name: kube-controller-manager
  namespace: kube-system
  resourceVersion: "3489"
  selfLink: /api/v1/namespaces/kube-system/endpoints/kube-controller-manager
  uid: 395decb9-a8fb-4c91-89a5-d31fb0bdfc0e
```

可以看到，当前的 leader 为k8s-01 节点。

现在停掉 k8s-01节点上的kube-controller-manager。

```shell
$ systemctl stop kube-controller-manager
$ systemctl status kube-controller-manager |grep Active
   Active: inactive (dead) since Fri 2020-04-24 13:48:36 CST; 46s ago
```

再查看一下当前的leader：

```shell
$ kubectl get endpoints kube-controller-manager --namespace=kube-system  -o yaml
apiVersion: v1
kind: Endpoints
metadata:
  annotations:
    control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"k8s-02_59ba72de-6138-475b-90e8-b2807cab5bbf","leaseDurationSeconds":15,"acquireTime":"2020-04-24T05:48:58Z","renewTime":"2020-04-24T05:49:28Z","leaderTransitions":1}'
  creationTimestamp: "2020-04-24T05:31:47Z"
  name: kube-controller-manager
  namespace: kube-system
  resourceVersion: "3802"
  selfLink: /api/v1/namespaces/kube-system/endpoints/kube-controller-manager
  uid: 395decb9-a8fb-4c91-89a5-d31fb0bdfc0e
```

可以看到现在的leader是k8s-02了。

##### 4）检查kube-proxy功能

查看ipvs路由规则

```shell
cat > deploy.sh << "EOF"
#!/bin/bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
    echo ">>> ${node_ip}"
    ssh root@${node_ip} "/usr/sbin/ipvsadm -ln"
done
EOF
```

输出结果如下：

```shell
$ ./deploy.sh 
>>> 192.168.0.71
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.254.0.1:443 rr
  -> 192.168.0.71:6443            Masq    1      0          0         
  -> 192.168.0.72:6443            Masq    1      0          0         
  -> 192.168.0.73:6443            Masq    1      0          0         
>>> 192.168.0.72
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.254.0.1:443 rr
  -> 192.168.0.71:6443            Masq    1      0          0         
  -> 192.168.0.72:6443            Masq    1      0          0         
  -> 192.168.0.73:6443            Masq    1      0          0         
>>> 192.168.0.73
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.254.0.1:443 rr
  -> 192.168.0.71:6443            Masq    1      0          0         
  -> 192.168.0.72:6443            Masq    1      0          0         
  -> 192.168.0.73:6443            Masq    1      0          0     
```

可见所有通过 https 访问 K8S SVC kubernetes 的请求都转发到 kube-apiserver 节点的 6443 端口。

##### 5）创建一个应用

查看节点状态

```shell
$ kubectl get nodes
NAME     STATUS   ROLES    AGE     VERSION
k8s-01   Ready    <none>   4h45m   v1.16.6
k8s-02   Ready    <none>   4h45m   v1.16.6
k8s-03   Ready    <none>   4h45m   v1.16.6
```

创建测试文件

```shell
cd /opt/k8s/work

cat > nginx-ds.yml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-ds
  labels:
    app: nginx-ds
spec:
  type: NodePort
  selector:
    app: nginx-ds
  ports:
  - name: http
    port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ds
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      app: nginx-ds
  template:
    metadata:
      labels:
        app: nginx-ds
    spec:
      containers:
      - name: my-nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
EOF
```

启动定义文件

**友情提醒：**可以先把上面的镜像pull下来。

```shell
$ kubectl create -f nginx-ds.yml
service/nginx-ds created
daemonset.apps/nginx-ds created
```

检查各节点的 Pod IP 连通性

```shell
$ kubectl get pods  -o wide -l app=nginx-ds
NAME             READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE   READINESS GATES
nginx-ds-25h8m   1/1     Running   0          61m   172.30.144.2   k8s-01   <none>           <none>
nginx-ds-7whgp   1/1     Running   0          61m   172.30.176.2   k8s-02   <none>           <none>
nginx-ds-9b85z   1/1     Running   0          61m   172.30.200.2   k8s-03   <none>           <none>
```

可以看到，nginx-ds的 Pod IP 分别是172.30.144.2、172.30.176.2、172.30.200.2。在所有 Node 上分别 ping 上面三个 Pod IP，看是否连通：

```shell
cat > deploy.sh << "EOF"
#!/bin/bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
    echo ">>> ${node_ip}"
    ssh ${node_ip} "ping -c 1 172.30.144.2"
    ssh ${node_ip} "ping -c 1 172.30.176.2"
    ssh ${node_ip} "ping -c 1 172.30.200.2"
done
EOF
```

检查服务 IP 和端口可达性

```shell
$ kubectl get svc -l app=nginx-ds  
NAME       TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-ds   NodePort   10.254.83.21   <none>        80:31573/TCP   66m
```

在所有 Node 上 curl Service IP：

```shell
cat > deploy.sh << "EOF"
#!/bin/bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
    echo ">>> ${node_ip}"
    ssh ${node_ip} "curl -s 10.254.83.21"
done
EOF
```

预期输出 nginx 欢迎页面内容。