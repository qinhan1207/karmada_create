# karmada操作

## 1.跨云多集群管理

### 1.1push模式

### 通过命令行工具注册集群[](https://karmada.io/zh/docs/userguide/clustermanager/cluster-registration#通过命令行工具注册集群)

使用以下命令将名为 `member1` 的集群注册到 Karmada 中。

```bash
karmadactl join member1 --kubeconfig=<karmada kubeconfig> --cluster-kubeconfig=<member1 kubeconfig>
```

### 通过命令行工具注销集群[](https://karmada.io/zh/docs/userguide/clustermanager/cluster-registration#通过命令行工具注销集群)

您可以使用以下命令注销集群。

```bash
karmadactl unjoin member1 --kubeconfig=<karmada kubeconfig> --cluster-kubeconfig=<member1 kubeconfig>

karmadactl unjoin k3s-cluster --kubeconfig=/home/qinhan/.kube/karmada-config --cluster-kubeconfig=/home/qinhan/k3s-for-karmada.yaml --cluster-context=default
```

在注销过程中，Karmada 分发到 `member1` 的资源会被清理。 并且，`--cluster-kubeconfig` 参数用于清理在 `join` 阶段创建的密钥。

重复此步骤以注销任何其他集群

## 2.多集群调度

### 1.1部署一个最简单的多集群deployment

目前有3个集群

```bash
qinhan@k8s-master01:~$ karmadactl get cluster
NAME           CLUSTER   VERSION        MODE   READY   AGE     ADOPTION
k3d-cluster1   Karmada   v1.31.5+k3s1   Push   True    115m    -
k3d-cluster2   Karmada   v1.31.5+k3s1   Push   True    114m    -
k3d-cluster3   Karmada   v1.31.5+k3s1   Push   True    7m20s   -
qinhan@k8s-master01:~$ 
```

#### 1.1.1创建一个propagationpolicy

```bash
vim propagationpolicy.yaml

apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy # 资源类型为：资源传播策略
metadata:
  name: example-policy # The default namespace is `default`.
spec:
  resourceSelectors: # 用于选择要传播的资源
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx # If no namespace is specified, the namespace is inherited from the parent object scope.
  placement:
    clusterAffinity:
      clusterNames: # 列出了要将资源传播到的集群名称
        - k3d-cluster1
        - k3d-cluster3

# 基于propagationpolicy.yaml来创建propagationPolicy
qinhan@k8s-master01:~/test$ karmadactl apply -f propagationpolicy.yaml 
propagationpolicy.policy.karmada.io/example-policy created
qinhan@k8s-master01:~/test$ karmadactl get propagationpolicy
NAME             CLUSTER   CONFLICT-RESOLUTION   PRIORITY   AGE   ADOPTION
example-policy   Karmada   Abort                 0          38s   -
qinhan@k8s-master01:~/test$ 
```

创建一个deployment

```bash
vim nginx-deploy.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21.6 # 使用一个稳定的 Nginx 版本
        ports:
        - containerPort: 80
        imagePullPolicy: IfNotPresent
```

使用创建的nginx-deploy.nginx来部署一个deployment

```bash
karmadactl apply -f nginx-deployment.yaml
```

执行完该操作后，可以在集群k3d-cluster1和k3d-cluster2看到创建的deployment

```bash
qinhan@ubuntu-host:~/k3d-config$ kubectl get deployment --kubeconfig=mycluster3.kubeconfig
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   3/3     3            3           24h
qinhan@ubuntu-host:~/k3d-config$ kubectl get deployment --kubeconfig=mycluster1.kubeconfig
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   3/3     3            3           24h
qinhan@ubuntu-host:~/k3d-config$ kubectl get deployment --kubeconfig=mycluster2.kubeconfig
No resources found in default namespace.
qinhan@ubuntu-host:~/k3d-config$ 
```

#### 1.1.2更新propagationpolicy

```yaml
kind: PropagationPolicy # 资源类型为：资源传播策略
metadata:
  name: example-policy # The default namespace is `default`.
spec:
  resourceSelectors: # 用于选择要传播的资源
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx # If no namespace is specified, the namespace is inherited from the parent object scope.
  placement:
    clusterAffinity:
      clusterNames: # 列出了要将资源传播到的集群名称
        - k3d-cluster2
```

应用更新后的propagationpolicy.yaml

```bash
karmadactl apply -f propagationpolicy.yaml
```

会看到刚才分配在集群k3d-cluster1和k3d-cluster2上的deployment消失了，而在k3d-cluster2上面分发了deployment

```bash
qinhan@ubuntu-host:~/k3d-config$ kubectl get deployment --kubeconfig=mycluster2.kubeconfig
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   3/3     3            3           85s
qinhan@ubuntu-host:~/k3d-config$ kubectl get deployment --kubeconfig=mycluster1.kubeconfig
No resources found in default namespace.
qinhan@ubuntu-host:~/k3d-config$ kubectl get deployment --kubeconfig=mycluster3.kubeconfig
No resources found in default namespace.
```

#### 1.1.3更新deployment

更新nginx-deply.yaml文件，将副本数调整为2并应用

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21.6 # 使用一个稳定的 Nginx 版本
        ports:
        - containerPort: 80
        imagePullPolicy: IfNotPresent
```

可以看到

```bash
# karmada控制平面
qinhan@k8s-master01:~/test$ karmadactl apply -f nginx-deploy.yaml 
deployment.apps/nginx configured
qinhan@k8s-master01:~/test$ karmadactl get deploy
NAME    CLUSTER   READY   UP-TO-DATE   AVAILABLE   AGE   ADOPTION
nginx   Karmada   2/2     2            2           24h   -
qinhan@k8s-master01:~/test$ 

# 查看k3d-cluster2集群
qinhan@ubuntu-host:~/k3d-config$ kubectl get deployment --kubeconfig=mycluster2.kubeconfig
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   2/2     2            2           9m47s
qinhan@ubuntu-host:~/k3d-config$ 
```

#### 1.1.4删除propagationpolicy.yaml

```bash
# karmada控制面板
qinhan@k8s-master01:~/test$ karmadactl delete propagationpolicy example-policy
propagationpolicy.policy.karmada.io "example-policy" deleted
qinhan@k8s-master01:~/test$ karmadactl get deployment
NAME    CLUSTER   READY   UP-TO-DATE   AVAILABLE   AGE   ADOPTION
nginx   Karmada   2/2     2            2           24h   -
qinhan@k8s-master01:~/test$ 

# k3d-cluster2集群
qinhan@ubuntu-host:~/k3d-config$ kubectl get deployment --kubeconfig=mycluster2.kubeconfig
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   2/2     2            2           12m
qinhan@ubuntu-host:~/k3d-config$ 
```

可以发现我们删除了propagationpolicy，但是已经分发给集群的资源并不会被删除，你需要在控制面板删除deployment

```bash
# karmada控制面板
qinhan@k8s-master01:~/test$ karmadactl delete deployment nginx
deployment.apps "nginx" deleted
qinhan@k8s-master01:~/test$ karmadactl get deployment
No resources found in default namespace.
qinhan@k8s-master01:~/test$ 
# 成员集群
qinhan@ubuntu-host:~/k3d-config$ kubectl get deployment --kubeconfig=mycluster2.kubeconfig
No resources found in default namespace.
qinhan@ubuntu-host:~/k3d-config$ 
```

### 1.2在 Karmada 中，将Deployment分发到指定的目标集群

#### LabelSelector

- **功能**：LabelSelector 是一种通过标签选择成员集群的过滤器。它使用 `metav1.LabelSelector` 类型。如果不为 nil 且不为空，则只有符合此过滤条件的集群会被选中。

- **配置示例**：

  ```yaml
  apiVersion: policy.karmada.io/v1alpha1
  kind: PropagationPolicy
  metadata:
    name: test-propagation
  spec:
    placement:
      clusterAffinity:
        labelSelector:
          matchLabels:
            location: us
  ```

- 或者使用 `matchExpressions`：

  ```yaml
  apiVersion: policy.karmada.io/v1alpha1
  kind: PropagationPolicy
  metadata:
    name: test-propagation
  spec:
    placement:
      clusterAffinity:
        labelSelector:
          matchExpressions:
          - key: location
            operator: In
            values:
            - us
  ```

#### FieldSelector

- **功能**：FieldSelector 是一种通过字段选择成员集群的过滤器。如果不为 nil 且不为空，则只有符合此过滤条件的集群会被选中。

- **配置示例**：

  ```yaml
  apiVersion: policy.karmada.io/v1alpha1
  kind: PropagationPolicy
  metadata:
    name: nginx-propagation
  spec:
    placement:
      clusterAffinity:
        fieldSelector:
          matchExpressions:
          - key: provider
            operator: In
            values:
            - huaweicloud
          - key: region
            operator: NotIn
            values:
            - cn-south-1
  ```

#### ClusterNames

- **功能**：明确指定集群名称。

- **配置示例**：

  ```yaml
  apiVersion: policy.karmada.io/v1alpha1
  kind: PropagationPolicy
  metadata:
    name: nginx-propagation
  spec:
    placement:
      clusterAffinity:
        clusterNames:
          - member1
          - member2
  ```

#### ExcludeClusters

- **功能**：排除特定集群。

- **配置示例**：

  ```yaml
  apiVersion: policy.karmada.io/v1alpha1
  kind: PropagationPolicy
  metadata:
    name: nginx-propagation
  spec:
    placement:
      clusterAffinity:
        exclude:
          - member1
          - member3
  ```

### 总结

通过这些配置，你可以灵活地控制资源在多个集群中的分发。根据你的需求选择合适的 `ClusterAffinity` 选项来实现目标。	

### 1.3Multiple cluster affinity groups(多集群亲和组)

用户可以在 `PropagationPolicy` 中设置 `ClusterAffinities` 字段，并声明多个集群亲和组。调度器会按照它们在规范中出现的顺序逐一评估这些组。不满足调度限制的组将被忽略，这意味着该组中的所有集群都不会被选择，除非某个集群也属于下一个组（一个集群可以属于多个组）。

如果没有任何组满足调度限制，则调度失败，这意味着不会选择任何集群，这种情况下，可能需要调整 `PropagationPolicy` 或集群配置，以确保至少一个组能满足要求。

注意：

- `ClusterAffinities` 不能与 `ClusterAffinity` 共存。
- 如果 `ClusterAffinity` 和 `ClusterAffinities` 都未设置，任何集群都可以作为调度候选。
- 潜在使用场景 1：本地数据中心的私有集群可以作为主组，集群提供商管理的集群可以作为次组。这样，Karmada 调度器会优先将工作负载调度到主组，只有在主组不满足限制（如资源不足）时才会考虑次组。

潜在使用场景 1：本地数据中心的私有集群可以作为主组，集群提供商管理的集群可以作为次组。这样，Karmada 调度器会优先将工作负载调度到主组，只有在主组不满足限制（例如资源不足）时，才会考虑次组。

潜在使用场景 2：在灾难恢复场景中，可以将集群组织为主组和备份组。工作负载会首先调度到主集群，当主集群出现故障（例如数据中心断电）时，Karmada 调度器可以将工作负载迁移到备份集群。

### 1.4Schedule based on Taints and Tolerations（基于污点和容忍度）

在 Karmada 中，您可以使用 `Taints` 和 `Tolerations` 来控制工作负载的调度行为。以下是一些关键点：

### Taints 和 Tolerations 的用法

#### 1. 添加 Taint 到集群

使用 `karmadactl` 为集群添加 Taint。例如：

```bash
karmadactl taint clusters foo dedicated=special-user:NoSchedule
```

这会在集群 `foo` 上添加一个 Taint，防止没有相应 Toleration 的工作负载调度到该集群。

#### 2. 在策略中添加 Toleration

要允许工作负载调度到带有特定 Taint 的集群，您需要在 `PropagationPolicy` 中指定相应的 Toleration：

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-propagation
spec:
  placement:
    clusterTolerations:
    - key: dedicated
      value: special-user
      effect: NoSchedule
```

### 支持的 Taint Effect

- **NoSchedule**: 不允许没有相应 Toleration 的新工作负载调度到该集群。
- **NoExecute**: 不仅阻止新工作负载调度，还会逐出现有的工作负载。

### 多集群故障转移

`NoExecute` Taints 可以用于多集群故障转移场景，确保在特定情况下自动迁移工作负载。

通过合理配置 Taints 和 Tolerations，您可以更好地控制工作负载在多集群环境中的调度策略。

### 1.5Multi region HA support（多区域高可用支持）

通过使用按区域分布的约束，您可以将工作负载分布到不同的区域，从而增强高可用性（HA）和灾难恢复能力。这样做有以下几个好处：

1. **地理冗余**：将工作负载分布在不同区域可以防范区域性故障。
2. **提高弹性**：如果某个区域出现问题，其他区域的工作负载仍能正常运行。
3. **降低延迟**：将工作负载部署在离用户更近的区域可以改善响应时间。
4. **法规遵从**：某些法规要求数据存储在特定区域，按区域分布可以帮助满足这些要求。

通过设置自动化策略，使工作负载在各区域间分布，可以实现一个稳健且具有弹性的部署策略。

要启用多区域部署，您可以使用以下命令自定义集群的区域设置：

你可以通过编辑集群配置来设置区域。以下是步骤：

1. 使用以下命令编辑集群配置：

   ```bash
   kubectl --kubeconfig ~/.kube/karmada-config edit cluster k3d-cluster1
   ```

2. 在 `spec` 部分中，添加或修改 `region` 字段：

   ```yaml
   spec:
     apiEndpoint: https://172.18.0.4:6443
     id: 257b5c81-dfae-4ae5-bc7c-6eaed9ed6a39
     impersonatorSecretRef:
       name: member1-impersonator
       namespace: karmada-cluster
     region: test
   ```

3. 保存并退出编辑器。

这样，您就为 `member1` 集群设置了 `region` 为 `test`。这可以帮助您在多区域部署中管理和调度资源。

1. 要将这个部署应用到你配置的多个区域，你需要确保已经创建了相应的 `PropagationPolicy`。以下是步骤：

2. 1. **检查并编辑 PropagationPolicy**：

      确保 `PropagationPolicy` 已正确配置，特别是 `resourceSelectors` 部分要匹配你的 `Deployment`。

      ```yaml
      apiVersion: policy.karmada.io/v1alpha1
      kind: PropagationPolicy
      metadata:
        name: nginx-propagation
      spec:
        resourceSelectors:
          - apiVersion: apps/v1
            kind: Deployment
            name: nginx
        placement:
          replicaScheduling:
            replicaSchedulingType: Duplicated
          spreadConstraints:
            - spreadByField: region
              maxGroups: 2
              minGroups: 2
            - spreadByField: cluster
              maxGroups: 1
              minGroups: 1
      ```

   2. 可以通过以下命令来查看具体分布情况

      ```
      karmadactl get work -A
      ```

### 1.6Multiple strategies of replica Scheduling（多副本调度策略）

