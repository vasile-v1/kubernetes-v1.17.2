## centos 7.5 部署 kubernetes-v1.17.2 + calico

环境：
CentOS Linux release 7.5.1804 \
kubernetes-v1.17.2\
calico v3.15.1

### 系统初始化

* 关闭防火墙，设置selinux，修改系统内核配置，及网络参数等

### k8s 相关系统设置,master和node节点均执行

* 修改主机名

* 添加hosts记录

* 关闭swap👇
```
[root@master01 ~]# swapoff -a
[root@master01 ~]# sed -i.bak '/swap/s/^/#/' /etc/fstab
```
* br_netfilter模块加载👇
```
[root@master01 ~]# lsmod | grep br_netfilter
[root@master01 ~]# modprobe br_netfilter
```
```
[root@master01 ~]# cat > /etc/rc.sysinit << EOF
#!/bin/bash\
for file in /etc/sysconfig/modules/*.modules ; do
[ -x $file ] && $file
done
EOF
[root@master01 ~]# cat > /etc/sysconfig/modules/br_netfilter.modules << EOF
modprobe br_netfilter
EOF
[root@master01 ~]# chmod 755 /etc/sysconfig/modules/br_netfilter.modules
```
* 内核参数修改👇
```
[root@master01 ~]# cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
[root@master01 ~]# sysctl -p /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```
* 设置kubernetes源
```
[root@master01 ~]# cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```
* dockers依赖
```
[root@master01 ~]# yum install -y yum-utils device-mapper-persistent-data lvm2 -y  #安装docker 依赖
```
* 设置docker 阿里源
```
[root@master01 ~]# yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```
* 清除yum缓存
```
[root@master01 ~]# yum clean all
[root@master01 ~]# yum makecache
```

### 软件安装
* 安装docker 
```
[root@master01 ~]# yum install docker-ce docker-ce-cli containerd.io -y 
```
配置docker镜像加速
```
[root@master01 ~]# mkdir -p   /etc/docker/
[root@master01 ~]# cat /etc/docker/daemon.json 
{"registry-mirrors": [
    "https://registry.docker-cn.com",
    "http://hub-mirror.c.163.com",
    "https://docker.mirrors.ustc.edu.cn"
  ],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
[root@master01 ~]# systemctl daemon-reload && systemctl start docker && systemctl enable docker
[root@master01 ~]# docker info  #确定是否有无错误
```
* 安装kubeadmin，kubelet，kubectl
```
[root@master01 ~]# yum list kubelet --showduplicates | sort -r | grep 1.17 #查看版本
kubelet.x86_64                       1.17.9-0                        kubernetes 
kubelet.x86_64                       1.17.8-0                        kubernetes 
kubelet.x86_64                       1.17.7-1                        kubernetes 
kubelet.x86_64                       1.17.7-0                        kubernetes 
kubelet.x86_64                       1.17.6-0                        kubernetes 
kubelet.x86_64                       1.17.5-0                        kubernetes 
kubelet.x86_64                       1.17.4-0                        kubernetes 
kubelet.x86_64                       1.17.3-0                        kubernetes 
kubelet.x86_64                       1.17.2-0                        kubernetes 
kubelet.x86_64                       1.17.2-0                        @kubernetes
kubelet.x86_64                       1.17.11-0                       kubernetes 
kubelet.x86_64                       1.17.1-0                        kubernetes 
kubelet.x86_64                       1.17.0-0                        kubernetes 
[root@master01 ~]# yum install -y kubelet-1.17.2 kubeadm-1.17.2 kubectl-1.17.2
[root@master01 ~]# systemctl enable kubelet && systemctl start kubelet
```

### 初始化master节点
* 下载k8s 镜像
```
[root@master01 ~]# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.17.2
[root@master01 ~]# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.17.2
[root@master01 ~]# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.17.2
[root@master01 ~]# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.17.2
[root@master01 ~]# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
[root@master01 ~]# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.3-0
[root@master01 ~]# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.5
[root@master01 ~]#
[root@master01 ~]# docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.17.2 k8s.gcr.io/kube-apiserver:v1.17.2
[root@master01 ~]# docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.17.2 k8s.gcr.io/kube-controller-manager:v1.17.2 
[root@master01 ~]# docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.17.2 k8s.gcr.io/kube-scheduler:v1.17.2 
[root@master01 ~]# docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.17.2 k8s.gcr.io/kube-proxy:v1.17.2 
[root@master01 ~]# docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1 k8s.gcr.io/pause:3.1
[root@master01 ~]# docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.3-0 k8s.gcr.io/etcd:3.4.3-0
[root@master01 ~]# docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.5 k8s.gcr.io/coredns:1.6.5 
[root@master01 ~]#
[root@master01 ~]# docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.17.2
[root@master01 ~]# docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.17.2
[root@master01 ~]# docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.17.2
[root@master01 ~]# docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.17.2
[root@master01 ~]# docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
[root@master01 ~]# docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.3-0
[root@master01 ~]# docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.5
```
* 初始化master
````
[root@master01 ~]# kubeadm init --kubernetes-version=1.17.2 --service-cidr=10.199.0.0/16 --pod-network-cidr=10.198.0.0/16 --ignore-preflight-errors=Swap
Your Kubernetes control-plane has initialized successfully!
To start using your cluster, you need to run the following as a regular user:
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/
Then you can join any number of worker nodes by running the following on each as root:
kubeadm join 192.168.10.246:6443 --token jtnq2z.ii4uwh7gzfesuddi \
    --discovery-token-ca-cert-hash sha256:b2ec56033ce03f1aa033f10fd80ec9c0beaec1d12b999dab3343489e0e912e46 
````

### master节点配置kubectl
```
[root@master01 ~]# mkdir -p $HOME/.kube
[root@master01 ~]# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@master01 ~]# chown $(id -u):$(id -g) $HOME/.kube/config
[root@master01 ~]# kubectl get node
NAME          STATUS   ROLES    AGE     VERSION
k8smaster01   Ready    master   6h46m   v1.17.2
```

### master节点安装kubectl 命令自动补全
```
[root@master01 ~]# yum -y install bash-completion
[root@master01 ~]# source /etc/profile.d/bash_completion.sh
```

### 部署 calico
* 安装Tigera Calico
```
[root@master01 ~]# kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
```
* 通过创建必要的自定义资源来安装Calico
```
[root@master01 ~]# curl  https://docs.projectcalico.org/manifests/custom-resources.yaml -o calico-resources.yaml
# 注意修改calico-resources.yaml 中的cidr 网段和前面配置一致
```

### 注册节点服务器
* 下载k8s必要镜像
```
[root@master01 ~]# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
[root@master01 ~]# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.17.2
[root@master01 ~]# docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.17.2 k8s.gcr.io/kube-proxy:v1.17.2 
[root@master01 ~]# docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1 k8s.gcr.io/pause:3.1
```
* 注册节点服务器
```
[root@node02 ~]# kubeadm join 192.168.10.246:6443 --token jtnq2z.ii4uwh7gzfesuddi \
  --discovery-token-ca-cert-hash sha256:b2ec56033ce03f1aa033f10fd80ec9c0beaec1d12b999dab3343489e0e912e46 
#默认token 过期时间为24 小时
```

### 修改kub-proxy 为ipvs 模式
```
[root@master01 ~]# kubectl get  cm -n kube-system  kube-proxy  -o yaml 
data.config.conf.mode: "ipvs" #将mode 设置为ipvs，并删除kubelet pod
```

### 部署kuubernetes dashboard
```
#github项目地址 https://github.com/kubernetes/dashboard
[root@master01 ~]# kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.3/aio/deploy/recommended.yaml
```
```
[root@master01 ~]# kubectl get po -n kubernetes-dashboard 
NAME                                        READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-c79c65bb7-p2c7r   1/1     Running   0          37h
kubernetes-dashboard-55fd8c78bd-rf9rv       1/1     Running   0          37h
[root@master01 ~]# kubectl get svc -n kubernetes-dashboard    
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP   10.199.104.101   <none>        8000/TCP        37h
kubernetes-dashboard        NodePort    10.199.111.223   <none>        443:30848/TCP   37h
#注意修改 service kubernetes-dashboard type 为Nodeport
```
获取登录token
```
[root@master01 ~]# kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

### 部署ingress-nginx
项目地址 https://kubernetes.github.io/ingress-nginx/
下面部署的是ingress-nginx 为版本v0.34.1 
```
[root@master01 ~]# wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.34.1/deploy/static/provider/cloud/deploy.yaml
```

### 部署kube-Prometheus
githuba 项目地址：https://github.com/prometheus-operator/kube-prometheus
```
kubectl create -f manifests/setup
kubectl create -f manifests/
```
### 注意事项
```
*上面步骤中docker 镜像是官方地址，很多镜像下载很慢或者无法下载。
*服务启动失败，pod无法启动，请查看日志。
```