# centos安装k8s

## 安装环境

```
Centos 7.9s
```

##  一．介绍

1．官网

```
 https://kubernetes.io/
 ```

2．必备工具安装

```
kubeadm、kubelet、 kubectl、Containerd 、helm
```

## 二．注意事项

- 1．kubeadm、kubelet、kubectl、Containerd 需要在每个pod机器上面添加。

- 2．Helm 只需要在master安装

## 三．安装

### 1．先通过linux系统中的包管理工具安装好必备软件

```
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/rpm/
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/rpm/repodata/repomd.xml.key
EOF
setenforce 0
yum install -y kubelet kubeadm kubectl
systemctl enable kubelet && systemctl start kubelet
```

### 2.Containerd安装

```
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y containerd.io
```

### 3.Helm 安装

```
https://get.helm.sh/helm-v3.14.2-linux-amd64.tar.gz

tar -zxvf helm-v3.14.2-linux-amd64.tar.gz

 mv linux-amd64/helm /usr/local/bin/helm
```

## 四．启动Kubernetes

### 1．启动master节点

（1）打开配置文件 /etc/containerd/config.toml 下找到此文件。如果配置文件为空或者没有直接执行

```
containerd config default > /etc/containerd/config.toml
```

（2）搜索配置文件中的SystemdCgroup 配置项，如果是false 改为true

### 2.关闭swap交换空间

```
swapoff -a
```

### 3.内核加载数据包转发模块

因为Linux 系统默认是禁止数据包转发功能。所以需要先执行下面命令


```
modprobe br_netfilter
```

### 4.设置主机的 IP Forward 功能

```
sysctl -w net.ipv4.ip_forward=1
```


### 5.关闭防火墙

```
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

### 6.修改crictl.yaml

```
cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 0
debug: false
pull-image-on-create: false
EOF
```

### 7.修改containerd配置文件

vim /etc/containerd/config.toml 搜索 sandbox_image 改为 registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9

### 8.重启containerd

```
sudo systemctl restart containerd
```

### 9.启动master 

```
kubeadm init --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers
```

如果一直没有响应，新开一个终端窗口执行,查看镜像列表

```
crictl images ls 
```

查看镜像是否拉取成功，如果没有拉取成功可以手动一个个拉取

```
 crictl pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.28.7
 crictl pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.28.7
 crictl pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.28.7
 crictl pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.28.7
 crictl pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9
 crictl pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.10-0
 crictl pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.10.1
```

成功后

master机器kubeadm init 成功后会返回类似下面的内容粘贴到node 机器执行
```
kubeadm join 192.168.0.212:6443 --token ri9xei.884o183yatzfb9e9 \
        --discovery-token-ca-cert-hash sha256:fac199221a73f1a24d4ec46e8cd1f440b4037ab2d106e864de55ac629c2cf964
```


要开始使用集群，您需要以普通用户身份运行以下命令：

```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

或者，如果您是 root 用户，则可以运行：
```
  export KUBECONFIG=/etc/kubernetes/admin.conf
```
### 10.安装从节点,Node节点

master机器kubeadm init 成功后会返回类似下面的内容粘贴到node 机器执行
```
kubeadm join 192.168.0.212:6443 --token ri9xei.884o183yatzfb9e9 \
        --discovery-token-ca-cert-hash sha256:fac199221a73f1a24d4ec46e8cd1f440b4037ab2d106e864de55ac629c2cf964
```

### 11、验证节点是否正常

执行下面命令查看STATUS 是否为Ready 状态 ，如果不是则为异常

```
kubectl get nodes
kubectl get pod -A 
```

执行kubectl get pod -A  发现coredns-*  一直是Pending 状态，判定确实网络cin 网络插件，需要安装calico，如下：

```
curl -o calico.yaml https://raw.githubusercontent.com/projectcalico/calico/release-v3.25/manifests/calico.yaml
kubectl apply -f calico.yaml
```

如果服务器服务访问githubusercontent.com ， 请在本机下载好 calico.yaml  上传至服务器

