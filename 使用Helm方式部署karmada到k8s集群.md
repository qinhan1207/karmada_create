# 使用Helm方式部署karmada到k8s集群

## 1.添加karmada chat repo

```bash
helm repo add karmada-charts https://raw.githubusercontent.com/karmada-io/karmada/master/charts
# 查看添加是否成功
helm repo list
# 搜索karmada相关版本
helm search repo karmada
```

## 2.安装karmada

```bash
# 创建命名空间
kubectl create namespace karmada-system

# 安装 Karmada（使用默认配置）
helm install karmada karmada-charts/karmada -n karmada-system --version v1.13.0

# 验证安装
kubectl get pods -n karmada-system
```

## 3.获取karmada的kubeconfig

```bash
# 提取 kubeconfig 并保存到文件
kubectl get secret -n karmada-system karmada-kubeconfig -o jsonpath='{.data.kubeconfig}' | base64 -d > karmada-config
```

## 4.下载对应版本的karmadactl

```bash
# 查看karmada版本
helm list -n karmada-system
# 下载
wget https://github.com/karmada-io/karmada/releases/download/v1.13.0/karmadactl-linux-amd64.tgz
tar -zxvf karmadactl-linux-amd64.tgz
sudo mv karmadactl /usr/local/bin/
# 验证版本一致性
karmadactl version
```

## 5.使用kubectl操作集群

```
kubectl get clusters --kubeconfig=/home/qinhan/.kube/karmada-config
```

### 5.1操作karmada时遇到域名解析问题时

```bash
qinhan@k8s-master01:~/.kube$ kubectl get clusters --kubeconfig=/home/qinhan/.kube/karmada-config
E0526 16:54:58.718306  139232 memcache.go:265] couldn't get current server API group list: Get "https://karmada-apiserver.karmada-system.svc.cluster.local:5443/api?timeout=32s": dial tcp: lookup karmada-apiserver.karmada-system.svc.cluster.local on 127.0.0.53:53: server misbehaving
E0526 16:54:58.718822  139232 memcache.go:265] couldn't get current server API group list: Get "https://karmada-apiserver.karmada-system.svc.cluster.local:5443/api?timeout=32s": dial tcp: lookup karmada-apiserver.karmada-system.svc.cluster.local on 127.0.0.53:53: server misbehaving
E0526 16:54:58.720577  139232 memcache.go:265] couldn't get current server API group list: Get "https://karmada-apiserver.karmada-system.svc.cluster.local:5443/api?timeout=32s": dial tcp: lookup karmada-apiserver.karmada-system.svc.cluster.local on 127.0.0.53:53: server misbehaving
E0526 16:54:58.720992  139232 memcache.go:265] couldn't get current server API group list: Get "https://karmada-apiserver.karmada-system.svc.cluster.local:5443/api?timeout=32s": dial tcp: lookup karmada-apiserver.karmada-system.svc.cluster.local on 127.0.0.53:53: server misbehaving
E0526 16:54:58.722972  139232 memcache.go:265] couldn't get current server API group list: Get "https://karmada-apiserver.karmada-system.svc.cluster.local:5443/api?timeout=32s": dial tcp: lookup karmada-apiserver.karmada-system.svc.cluster.local on 127.0.0.53:53: server misbehaving
Unable to connect to the server: dial tcp: lookup karmada-apiserver.karmada-system.svc.cluster.local on 127.0.0.53:53: server misbehaving
```

### 5.1.1 **编辑 systemd-resolved 配置文件**

```bash
sudo vi /etc/systemd/resolved.conf
```

### 5.1.2**修改 DNS 配置**

```ini
[Resolve]
DNS=10.0.0.10 8.8.8.8
Domains=~cluster.local svc.cluster.local services.cluster.local
LLMNR=no
MulticastDNS=no
DNSSEC=no  # 禁用 DNSSEC 验证（可选，避免兼容性问题）
```

### 5.1.3**重启 systemd-resolved 服务**

```bash
sudo systemctl restart systemd-resolved
sudo systemd-resolve --flush-caches
```

