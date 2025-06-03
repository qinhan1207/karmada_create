# 使用k3d来模拟多个集群

## 1.安装依赖工具

```bash
# 安装 Docker（k3d 依赖 Docker）
curl -fsSL https://get.docker.com | sh
sudo systemctl enable --now docker

# 安装 k3d（K3s 轻量化集群工具）
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

# 安装 kubectl（Kubernetes 命令行工具）
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

## 2.修改 `k3s` 的配置文件，**为它的 `containerd` 显式设置镜像加速器或者代理**

### 2.1在宿主机创建一个 `registries.yaml` 文件

```bash
mkdir -p ~/.k3d/registries
vim ~/.k3d/registries/registries.yaml
```

填入以下内容（使用国内镜像源）：

```yaml
mirrors:
  "docker.io":
    endpoint:
      - "https://hub-mirror.c.163.com"
      - "https://mirror.baidubce.com"
      - "https://registry.aliyuncs.com"
```

## 2.2创建集群时挂载这个配置文件

```bash
# 创建集群1
k3d cluster create mycluster \
  --api-port 192.168.157.15:6443 \
  --volume ~/.k3d/registries/registries.yaml:/etc/rancher/k3s/registries.yaml
 
# 创建集群2
k3d cluster create mycluster2 \
  --api-port 192.168.157.15:6444 \
  --volume ~/.k3d/registries/registries.yaml:/etc/rancher/k3s/registries.yaml
```

## 3.导出kubeconfig

```bash
k3d kubeconfig get mycluster1 > mycluster1.kubeconfig
k3d kubeconfig get mycluster2 > mycluster2.kubeconfig
```

然后将这两个文件 **复制到 Karmada 控制面虚拟机上**，比如通过 `scp`：

```bash
scp -r k3d-config/ qinhan@<目标主机IP>:/home/qinhan/
	-r：递归复制整个文件夹（你如果是目录就必须加）

	k3d-config/：你要复制的目录（比如里面有 mycluster1.kubeconfig）

	qinhan@<目标主机IP>：用户名@目标机器的 IP 地址，比如 192.168.56.101

	:/home/qinhan/：你目标主机上的存放路径
```

## 4.注册集群到karmada

```bash
# 在安装了karmada的集群执行
karmadactl join cluster1-name \
  --cluster-kubeconfig=/path/to/cluster1-kubeconfig.yaml \
  --kubeconfig=/home/qinhan/.kube/karmada-config
  
  
  --cluster-kubeconfig：
  	用于指定要加入的目标集群的 kubeconfig 文件。
  	这个文件包含该集群的访问和认证信息。
  	
  --kubeconfig：
  	用于指定 Karmada 控制平面的 kubeconfig 文件。
	这个文件包含访问 Karmada 控制平面的必要信息。
```

## 5.手动导入镜像到k3d创建的集群

```bash
k3d image import nginx:1.21.6 -c mycluster
	
```

