# 通过kind工具来搭建karmada开发测试环境

## 前提条件

### 1.安装docker

```bash
qinhan@qinhan-VMware-Virtual-Platform:~$ docker -v
Docker version 28.1.1, build 4eba377
```

### 2.安装go

```bash
qinhan@qinhan-VMware-Virtual-Platform:~$ go version
go version go1.23.8 linux/amd64
```

### 3.安装kubectl

```bash
qinhan@qinhan-VMware-Virtual-Platform:~$ kubectl version
Client Version: v1.33.0
Kustomize Version: v5.6.0
Server Version: v1.31.2
WARNING: version difference between client (1.33) and server (1.31) exceeds the supported minor version skew of +/-1

```

### 4.安装kind

```bash
qinhan@qinhan-VMware-Virtual-Platform:~$ kind version
kind v0.27.0 go1.23.8 linux/amd64
```

### 5.安装 Karmadactl

```bash
curl -s https://raw.githubusercontent.com/karmada-io/karmada/master/hack/install-cli.sh | sudo bash
```

### 6.安装kubectl-karmada

```bash
curl -s https://raw.githubusercontent.com/karmada-io/karmada/master/hack/install-cli.sh | sudo bash -s kubectl-karmada
```

## 安装karmada

### 1.使用kind创建集群（首先确保已经将karmada克隆到本地）

```
cd karmada
// 创建一个名为host的主集群
hack/create-cluster.sh host $HOME/.kube/host.config
```

### 2.将集群加入到上下文

```bash
kind export kubeconfig --name host --kubeconfig ~/.kube/config
```

### 3.查看集群所有上下文并切换集群

```bash
# 查看集群所有上下文
kubectl config get-contexts

# 切换集群
kubectl config use-context kind-host
```

### 4. **获取 `karmada-host` 集群的 kubeconfig(如果）**

默认情况下，`kind` 集群的 kubeconfig 会合并到 `~/.kube/config` 中，但我们可以单独导出 `karmada-host` 集群的配置：

```
# 导出 karmada-host 集群的 kubeconfig 到 host.config
kind get kubeconfig --name kind-host > ~/.kube/host.config
```

验证文件是否生成：

```
ls ~/.kube/host.config
```

### 5.遇到webhook组件报too many files的错误（可提前执行，防止报错）

```bash
# 进入到运行webhook组件的容器的内部并执行下面命令
docker exec -it host-control-plane bin/bash
sysctl fs.inotify.max_user_watches=16384
sysctl -w fs.inotify.max_user_watches=100000
sysctl -w fs.inotify.max_user_instances=100000
```

### 6. **安装 Karmada 并确认版本**

确保已安装与你的 Karmada 版本匹配的 CLI（以 v1.13.2 为例）：

```bash
sudo kubectl karmada init --crds https://github.com/karmada-io/karmada/releases/download/v1.13.2/crds.tar.gz --kubeconfig=$HOME/.kube/host.config
```

### 7.手动导入镜像到kind创建的集群 （若网络不好，可在安装前执行）

```bash
docker pull docker.io/alpine:3.21.0
kind load docker-image docker.io/alpine:3.21.0 --name karmada-host

docker pull registry.k8s.io/etcd:3.5.16-0
kind load docker-image registry.k8s.io/etcd:3.5.16-0 --name karmada-host

docker pull registry.k8s.io/kube-apiserver:v1.31.3
kind load docker-image registry.k8s.io/kube-apiserver:v1.31.3 --name karmada-host

docker pull docker.io/karmada/karmada-controller-manager:v1.9.0
kind load docker-image docker.io/karmada/karmada-controller-manager:v1.9.0 --name karmada-host

docker pull docker.io/karmada/karmada-aggregated-apiserver:v1.13.2
kind load docker-image docker.io/karmada/karmada-aggregated-apiserver:v1.13.2 --name karmada-host

docker pull docker.io/karmada/karmada-scheduler:v1.13.2
kind load docker-image docker.io/karmada/karmada-scheduler:v1.13.2 --name karmada-host

docker pull registry.k8s.io/kube-controller-manager:v1.31.3
kind load docker-image registry.k8s.io/kube-controller-manager:v1.31.3 --name karmada-host

docker pull docker.io/karmada/karmada-controller-manager:v1.13.2
kind load docker-image docker.io/karmada/karmada-controller-manager:v1.13.2 --name karmada-host

docker pull docker.io/karmada/karmada-webhook:v1.13.2
kind load docker-image docker.io/karmada/karmada-webhook:v1.13.2 --name karmada-host

# 批量拉取
docker pull docker.io/alpine:3.21.0 &
docker pull registry.k8s.io/etcd:3.5.16-0 &
docker pull registry.k8s.io/kube-apiserver:v1.31.3 &
docker pull docker.io/karmada/karmada-controller-manager:v1.9.0 &
docker pull docker.io/karmada/karmada-aggregated-apiserver:v1.13.2 &
docker pull docker.io/karmada/karmada-scheduler:v1.13.2 &
docker pull registry.k8s.io/kube-controller-manager:v1.31.3 &
docker pull docker.io/karmada/karmada-controller-manager:v1.13.2 &
docker pull docker.io/karmada/karmada-webhook:v1.13.2 &

kind load docker-image --name=host \
  docker.io/alpine:3.21.0 \
  registry.k8s.io/etcd:3.5.16-0 \
  registry.k8s.io/kube-apiserver:v1.31.3 \
  docker.io/karmada/karmada-controller-manager:v1.9.0 \
  docker.io/karmada/karmada-aggregated-apiserver:v1.13.2 \
  docker.io/karmada/karmada-scheduler:v1.13.2 \
  registry.k8s.io/kube-controller-manager:v1.31.3 \
  docker.io/karmada/karmada-controller-manager:v1.13.2 \
  docker.io/karmada/karmada-webhook:v1.13.2
```

### 8.如果安装失败则卸载并重新安装（执行5.重新进行安装）

```bash
karmadactl deinit --kubeconfig=$HOME/.kube/host.config
```

### 9.安装成功后

```
安装成功后通过指定用户目录下的.karmada/karmada-apiserver.config配置文件来操作karmada
```

### 10.创建一个成员集群并将其加入kubectl上下文再加入karmada集群

```bash
# 进入karmada目录
cd karmada
# 创建集群
hack/create-cluster.sh member1 $HOME/.kube/member1.config
# 加入上下文
kind export kubeconfig --name member1 --kubeconfig ~/.kube/config
# 查看上下文
kubectl config get-contexts
# 切换上下文
kubectl config use-context kind-host

```

### 11.将成员注册到karmada

#### **1. Push 模式（推荐）**

**适用场景**：成员集群可以直接访问 Karmada 控制平面。

```bash
# 切换到主集群
kubectl config use-context kind-host
# 将成员集群注册到karmada
sudo kubectl karmada join member1 \
  --cluster-kubeconfig=$HOME/.kube/member1.config \
  --kubeconfig=/etc/karmada/karmada-apiserver.config  # 注意参数名改为 --kubeconfig
  
# 验证注册结果
sudo kubectl --kubeconfig=/etc/karmada/karmada-apiserver.config get clusters

```

#### **2. Pull 模式**

**适用场景**：Karmada 控制平面无法直接访问成员集群（如跨网络环境）。

### 12.方便后续操作

```bash
# 将karmada配置合并到默认kubeconfig
sudo cp /etc/karmada/karmada-apiserver.config ~/.kube/
sudo chown $USER:$USER ~/.kube/karmada-apiserver.config
export KUBECONFIG=~/.kube/config:~/.kube/karmada-apiserver.config
kubectl config view --flatten > /tmp/config && mv /tmp/config ~/.kube/config

# 之后可以直接使用
kubectl --context=karmada-apiserver get clusters
```

