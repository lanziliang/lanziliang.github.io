---
title: VMware搭建k8s集群
date: 2023-09-10 21:23:15
categories:
- Kubernetes
tags: 
- k8s
---

# 规划

- 虚拟机

  | 主机名     | ip              | 配置                     | 系统            |
  | ---------- | --------------- | ------------------------ | --------------- |
  | k8s-master | 192.168.101.128 | CPU:2核 内存:2G 磁盘:20G | CentOS 7.9.2009 |
  | k8s-node1  | 192.168.101.129 | CPU:2核 内存:2G 磁盘:20G | CentOS 7.9.2009 |
  | k8s-node1  | 192.168.101.130 | CPU:2核 内存:2G 磁盘:20G | CentOS 7.9.2009 |

- 版本

  | 包名       | 版本   |
  | ---------- | ------ |
  | Docker     | 24.0.5 |
  | Kubernetes | 1.28.0 |

# 创建虚拟机

- 下载VMware [官网地址](https://www.vmware.com/products.html?resource=product-listing:anywhere-workspace/desktop-hypervisor#product-listing:anywhere-workspace/desktop-hypervisor)
- 下载CentOS 7.9.2009 [镜像地址](https://mirrors.tuna.tsinghua.edu.cn/centos/7.9.2009/isos/x86_64/)

# 安装 Docker

> 参考文档：https://docs.docker.com/engine/install/centos/

1. 移除已有docker相关包

   ```
   yum remove docker \
       docker-client \
       docker-client-latest \
       docker-common \
       docker-latest \
       docker-latest-logrotate \
       docker-logrotate \
       docker-engine
   ```

2. 配置yum源

   ```
   yum install -y yum-utils
   yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
   ```

3. 安装Docker Engine

   ```
   yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   ```

4. 启动

   ```
   systemctl enable docker --now
   ```

5. 配置加速及cgroup驱动

   ```
   tee /etc/docker/daemon.json <<-'EOF'
   {
       "registry-mirrors": [
           "https://docker.mirrors.ustc.edu.cn",
           "https://hub-mirror.c.163.com",
           "https://reg-mirror.qiniu.com",
           "https://registry.docker-cn.com"
       ],
       "exec-opts": ["native.cgroupdriver=systemd"],
       "log-driver": "json-file",
       "log-opts": {
           "max-size": "100m"
       },
       "storage-driver": "overlay2"
   }
   EOF
   
   systemctl daemon-reload
   systemctl restart docker
   ```

# 安装 cri-dockerd

```
ARCH="amd64"
CRI_DOCKERD_VERSION="0.3.4"
wget https://github.com/Mirantis/cri-dockerd/releases/download/v${CRI_DOCKERD_VERSION}/cri-dockerd-${CRI_DOCKERD_VERSION}.${ARCH}.tgz
tar -xf cri-dockerd-${CRI_DOCKERD_VERSION}.${ARCH}.tgz
install -o root -g root -m 0755 cri-dockerd/cri-dockerd /usr/local/bin/cri-dockerd

wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/v${CRI_DOCKERD_VERSION}/packaging/systemd/cri-docker.service
wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/v${CRI_DOCKERD_VERSION}/packaging/systemd/cri-docker.socket
install cri-docker.service /etc/systemd/system/cri-docker.service
install cri-docker.socket /etc/systemd/system/cri-docker.socket
sed -i -e 's,/usr/bin/cri-dockerd --container-runtime-endpoint fd://,/usr/local/bin/cri-dockerd --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.8 --container-runtime-endpoint fd://,' /etc/systemd/system/cri-docker.service

systemctl daemon-reload
systemctl enable cri-docker.service
systemctl enable --now cri-docker.socket
```

# 使用 kubeadm 引导安装集群

## 基础环境 (所有机器)

- 关闭防火墙

  ```
  systemctl stop firewalld
  systemctl disable firewalld
  ```

- 关闭selinux

  ```
  setenforce 0
  sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
  ```

- 关闭swap

  ```
  swapoff -a  
  sed -ri 's/.*swap.*/#&/' /etc/fstab
  ```

- 设置主机名和hosts

  ```
  hostnamectl set-hostname xxxx
  cat <<EOF>> /etc/hosts
  192.168.101.128     k8s-master
  192.168.101.129     k8s-node1
  192.168.101.130     k8s-node2
  EOF
  ```

- 允许 iptables 检查桥接流量

  ```
  cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
  br_netfilter
  EOF
  
  cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  EOF
  
  sysctl --system
  ```

- 时间同步

  ```
  timedatectl set-timezone Asia/Shanghai
  yum install ntpdate -y
  ntpdate time.windows.com
  ```

## 所有节点安装kubelet、kubeadm、kubectl

```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
    http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

sudo yum install -y kubelet-1.28.0 kubeadm-1.28.0 kubectl-1.28.0 --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```

## 初始化主节点

```
# 所有机器添加master域名映射，以下需要修改为自己的ip
echo "192.168.101.128  cluster-endpoint" >> /etc/hosts

#主节点初始化
kubeadm init \
--apiserver-advertise-address=192.168.101.128 \
--image-repository registry.aliyuncs.com/google_containers \
--control-plane-endpoint=cluster-endpoint \
--kubernetes-version v1.28.0 \
--cri-socket unix:///var/run/cri-dockerd.sock \
--service-cidr=10.96.0.0/16 \
--pod-network-cidr=172.31.0.0/16

#所有网络范围不重叠
```

## 设置./kube/config

```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

## 安装网络组件

```
wget https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
kubectl apply -f calico.yaml
```

## 加入节点

- 运行 `kubeadm init` 输出的命令

  ```
  kubeadm join cluster-endpoint:6443 --token rggsd3.yczrwviph9f49ld7 \
  --cri-socket unix:///var/run/cri-dockerd.sock \
  --discovery-token-ca-cert-hash sha256:9f7b737692ef243eaa4f618d50b5c5abbfd3decbc48506a19886b68dbda54364
  ```

- 如果没有记住 `kubeadm init` 的输出，通过一下命令获取令牌

  ```
  # 列出当前可用令牌
  kubeadm token list
  
  # 新建
  kubeadm token create
  ```

- 如果没有 `--discovery-token-ca-cert-hash` 的值，执行下面命令获取

  ```
   openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
  openssl dgst -sha256 -hex | sed 's/^.* //'
  ```

- 在mater节点运行 `kubectl get nodes` 验证