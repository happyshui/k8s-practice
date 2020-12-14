# kubernetes

# Pod

## Basic

进程与进程组: Container and Pod

一个逻辑单位，多个容器的组合；kubernetes的原子调度单位

Task co-scheduling Problem

- Google Omega: 乐观调度处理冲突
- Kubernetes: Pod

## 实现机制

1. 共享网络
    1. 通过Infra Container的方式共享同一个Network Namespace
        - 其他所有容器都会通过 Join Namespace 的方式加入到 Infra container 的 Network Namespace 中
        - image: k8s.gcr.io/pause; by Assembly language; size: 100-200KB
    2. 直接使用localhost通信
    3. 网络设备所有容器看到的均一致
    4. 一个Pod只有一个IP地址，即Pod的Network Namespace对应的IP地址
    5. 整个Pod里面，Infra container 第一个启动，并且整个Pod生命周期等同于 Infra container 的生命周期的，而与其他容器无关
2. 共享存储

## 设计模式

所有“设计模式”的本质都是: 解耦和重用

### InitContainer

### Sidecar

- 日志收集
- 代理容器
- 适配器容器

# 应用编排与管理

### metadata

- Labels
- Annotations
- Ownereference

### 控制器模式

[](https://www.bilibili.com/video/BV1nE411f7qR)

11:45 - 17:00 需要温习

- 声明式的API驱动 - K8S资源对象
- 由控制器异步的控制系统向终态驱近
- 便于扩展 - 自定义资源和控制器

### Deployment

```bash
$ kubectl set image deployment/nginx nginx=nginx:1.9.1
$ kubectl rollout undo deployment/nginx
$ kubectl rollout undo deployment/nginx --to-revision=2 # 回滚到指定版本
$ kubectl rollout history deployment/nginx # 查看历史版本
# deployment revisionHistoryLimit 保留的版本数量 默认10
```

- Deployment负责管理不同版本的ReplicaSet，由ReplicaSet管理Pod副本数
- 每个ReplicaSet对应一个Deployment template的一个版本
- 一个ReplicaSet的Pod版本都相同

Deployment Controller

### Job

- restartPolicy: 重启策略
- backoffLimit: 重试次数限制
- completions: 代表本pod队列执行次数
- parallelism: 代表并行执行个数

### CronJob

- schedule： 定时类似crontab
- startingDeadlineSeconds: Job最长启动时间
- concurrencyPolicy: 是否允许并行运行
- successfulJobsHistoryLimit: 允许留存历史的Job个数

### 管理模式

- Job Controller负责根据配置创建Pod
- 跟踪Job状态，根据配置及时重试Pod或继续创建

### DeamonSet

守护进程控制器

## StatefulSet

- 每个Pod有Order序号，会按序号创建、删除、更新Pod
- 通过配置headless service，使每个Pod有一个唯一的网络标识（hostname）
- 通过配置PVC Template，每个Pod有一块独享的PV存储盘
- 支持一定数量的灰度发布

### 管理模式

- ControllerRevision: 管理不同版本的Template模版
- PVC: volumeClaimTemplates
- Pod: 顺序创建、删除、更新，每个Pod有唯一的序号

**扩缩容管理策略**

podManagementPolicy

- OrderedReady: 扩容按照order顺序执行，必须前面的Pod状态ready了，继续下一个扩；缩容时，倒序删除
- Parallel: 并行扩缩容，不需要等前面Pod都ready或删除再处理下一个

**升级策略**

- RollingUpdate - 滚动升级
- OnDelete - 禁止主动升级

# 应用配置

## ConfigMap

```yaml
# 环境变量
valueFrom:
  configMapKeyRef:
    name: xxx
    key: xxx

# 挂载方式
volumeMounts:
- name: xxx
  mountPath: xxx

volumes:
  - name: xxx
    configMap:
      name: xxx
```

注意点:

- 文件大小限制: 1MB
- Pod只能引用相同的Namespace的ConfigMap
- Pod引用的ConfigMap不存在时，Pod无法创建成功
- 只有通过API创建的Pod才可以使用ConfigMap

## Secret

注意点:

- 文件大小限制: 1MB
- base64编码；可以考虑结合Kubernetes + Vault解决敏感信息的加密和权限管理
- 最佳实践: 不要使用list/watch；推荐使用GET获取

## ServiceAccount

## Resources.requests/limits

- CPU
- Memory
- Ephemeral storage（临时存储）

QoS

- Guaranteed
    - requests and limit
- Burstable
    - requests
- BestEffort
    - None

驱逐顺序: BestEffort Burstable Guaranteed的顺序驱逐Pod

## SecurityContext

限制容器的行为，从而保障系统和其他容器的安全

1. 容器级别: 仅对容器生效
2. Pod级别: 对Pod内的所有容器生效
3. Pod Security Policies(PSP): 对集群内的所有Pod生效

## InitContainers

InitContainer VS Container

1. InitContainer会优先于普通的Container启动执行，直到所有的Init执行完成后，普通的Container才会被启动
2. Pod中多个InitContainer之间是按次序依次启动执行，而Pod中多个普通Container是并行启动
3. InitContainer执行成功后即结束退出

用途: 用于初始化或一些启动的前置条件检验

# 应用存储和持久化数据卷

Volume类型

- 本地存储: emptydir / hostpath ...
- 网络存储:
    - in-tree: awsElsticblockStore / gcePersistentDisk / nfs / GlusterFS / CephFS
    - out-of-tree: flexvolume/csi等网络存储plugins
- Projected volume: secret / configmap ...
- PV/PVC

## Persistent Volumes

存储和计算分离

PV/PVC设计意图:

1. 职责分离
2. PVC简化了User对存储的需求，PV才是存储的实际信息的承载体
3. PVC像是面向对象编程中抽象出来的接口，PV是接口对应的实现

### Static Volume Provisioning

### Dynamic Volume Provisioning

- StorageClass

PV状态流转

Create PV - - > pending - - > available - - > bound - - > released - - > deleted (OR failed)

说明: 到达released状态的PV无法根据Reclaim Policy回到available状态再次bound新的PVC

想复用原来PV对应的存储中的数据，两种方式:

1. 复用old PV中记录的存储信息新建PV对象
2. 直接从PVC对象复用，即不unbound PVC和PV（即: StatefulSet处理存储状态的原理）

## 存储快照 - Snapshot

## 存储拓扑调度

Local PV: delay binding

## 架构与插件使用

[](https://www.bilibili.com/video/BV1BJ411y7ta)

![kubernetes%208cb51f8e78ef4e5fbef90eaac39d554b/Screen_Shot_2020-12-07_at_8.05.19_PM.png](kubernetes%208cb51f8e78ef4e5fbef90eaac39d554b/Screen_Shot_2020-12-07_at_8.05.19_PM.png)

### Flexvolume

### CSI

# 观测性

## Liveness 存活探针

- 检测失败: 杀掉Pod
- 适用场景: 重新拉起的应用

## Readiness 就绪探针

- 检测失败: 切断上层流量到Pod
- 适用场景: 启动后无法立即对外服务的应用

探测方式

- httpGet
- Exec
- tcpSocket

探测结果

- Success
- Failure
- Unknown

重启策略

- Always
- OnFailure
- Never

重要参数

- initialDelaySeconds
- periodSeconds
- timeoutSeconds
- successThreshold
- failureThreshold

调试工具

- kubectl-debug

## 监控与日志

### 监控

- 资源监控
- 性能监控
- 安全监控
- 事件监控

Heapster: 早期的版本 - - > Metrics-Server: 新的版本

**Heapster**

![kubernetes%208cb51f8e78ef4e5fbef90eaac39d554b/Screen_Shot_2020-12-03_at_8.16.27_PM.png](kubernetes%208cb51f8e78ef4e5fbef90eaac39d554b/Screen_Shot_2020-12-03_at_8.16.27_PM.png)

**Metrics-Server**

![kubernetes%208cb51f8e78ef4e5fbef90eaac39d554b/Screen_Shot_2020-12-03_at_8.17.04_PM.png](kubernetes%208cb51f8e78ef4e5fbef90eaac39d554b/Screen_Shot_2020-12-03_at_8.17.04_PM.png)

[监控接口标准](https://www.notion.so/d39c71f8b04549c585b42990f99aea46)

**kube-eventer**

[AliyunContainerService/kube-eventer](https://github.com/AliyunContainerService/kube-eventer.git)

# 网络

[](https://www.bilibili.com/video/BV1XJ411s7cd)

三个基本条件

- 任意两个Pod可以相互通信
- Node和Pod可以直接通信
- Pod可见的IP地址确为其他Pod与其通信是所用的一致

四大目标

- 容器与容器间的通信
- Pod与Pod之间的通信
- Pod与Servcie之间的通信
- 外部与Service之间的通信

容器网络方案:

- Underlay
- Overlay

差异: 在于是否与Host网络同层

## Network namespace - Netns

实现网络虚拟化的内核基础，创建隔离的网络空间

每个Pod拥有独立的Netns空间，Pod内的Container共享该空间，可通过Loopback接口实现通信

主流的网络实现方案

- Flannel
- Calico: BGP，对底层网络有要求
- Cannal
- Cilium: 基于eBPF和XDP的高性能Overlay网络方案
- ...

## Network Policy

基于策略的网络控制，用于隔离应用

要决定三件事

- 控制对象: 通过spec字段，podSelector等条件筛选
- 流方向: Ingress（入Pod流量），Egress（出Pod流量）
- 流特征: 对端（name/pod Selector），IP段（ipBlock），Protocol，Port

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
```

## Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: test
  labels:
    app: test
spec:
  selector:
    app: test
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
```

ClusterIP 

Headless Service

- clusterIP: None
- 通过DNS的A记录解析访问对应的地址

NodePort

LoadBalancer

## CNI

- Container Network Interface
- kubelet通过这个标准的API调用不同的网络插件实现配置网络
- CNI插件: 一系列实现了CNI API接口的网络插件

### 三种实现模式

- Overlay
- 路由
- Underlay

### 如何选择适合的

1. 环境限制
    1. 虚拟化
    2. 物理机
    3. 公有云 
2. 功能需求
    1. 安全
    2. 集群外资源互联互通
    3. 服务发现与负载均衡
3. 性能需求
    1. Pod创建速度
    2. Pod网络性能

自己开发CNI

19:00-26:00

[](https://www.bilibili.com/video/BV1XJ411W7zZ)

# 容器

**容器的本质: 一个进程，是一个视图被隔离，资源受限的进程**

由于容器实际上是一个"单进程"模型，容器里面最好不要启动多个进程

### Namespace

- mount
- uts
- pid
- network
- user
- ipc
- cgroup
    - disable cgroup namespace
    - enable cgroup namespace

## Cgroup

- systemd cgroup driver
- cgroupfs cgroup driver

Docker常用的cgroup

- CPU
- MEM
- device
- freezer
- blkio
- PID

Runc还支持的cgroup

- net_cls
- net_prio
- hugetlb
- perf_event

## Images

- 基于联合文件系统
- 不同的层可以被其他的镜像复用
- 容器的可写层可以做成镜像新的一层

EX: overlay

![kubernetes%208cb51f8e78ef4e5fbef90eaac39d554b/Screen_Shot_2020-12-03_at_11.23.20_PM.png](kubernetes%208cb51f8e78ef4e5fbef90eaac39d554b/Screen_Shot_2020-12-03_at_11.23.20_PM.png)

- mergedir: 整合了lower层和upper读写层显示出来的视图
- upper: 容器读写层
- workdir: 类似中间层，对upper层的写入，先写入workdir，再写入upper层
- lower: 镜像层

## Containerd容器架构

[](https://www.bilibili.com/video/BV1RJ411m7dT)

18:00 - 26:00

## CRI - container runtime interface

- 抽象一套Pod级别的容器接口，解耦kubelet与容器运行时
- 定义通过gRPC协议通讯

### CRI 设计

![kubernetes%208cb51f8e78ef4e5fbef90eaac39d554b/Screen_Shot_2020-12-08_at_7.56.56_PM.png](kubernetes%208cb51f8e78ef4e5fbef90eaac39d554b/Screen_Shot_2020-12-08_at_7.56.56_PM.png)

### CRI 实现

- CRI-containerd
- CRI-O
- PouchContainer
- ...

cri-tools: 测试工具

## 安全容器

[](https://www.bilibili.com/video/BV1VE411471i)

- Kata Containers
- gVisor

## RuntimeClass与使用多容器运行时

[](https://www.bilibili.com/video/BV1LE41147jG)

- Docker
- RKT
- Kata & gVisor

```go
type RuntimeClass struct {
    metav1.TypeMeta
    metav1.ObjectMeta
    
    Handler string          // 对应CRI配置中的某个容器运行时的handler
    Overhead *Overhead      // Pod的额外资源开销
    Scheduling *Scheduling  // 用于把Pod调度到支持该RuntimeClass的节点
}
```

### 总结

- RuntimeClass是Kubernetes一种内置的全局域资源，主要用来解决多个容器运行时混用的问题
- RuntimeClass中配置Scheduling，可以让Pod自动调度到指定的容器运行时的节点上
- RuntimeClass中配置Overhead，可以把Pod中业务运行所需以外的开销统计进来，让调度、ResourceQuota、kubelet Pod驱逐等行为更准确

# ETCD

## 架构

一般由3或5个节点组成，使用Raft算法完成一致性

quorum = (n+1)/2

## 使用场景

- 服务发现
    - 资源注册
    - 存活性检测
    - API Gateway无状态，可以水平扩展
    - 支持上万个进程的规模
- 分布式系统设计模式: Leader election （Master选主）
- 分布式系统并发控制
    - 分布式信号量
    - 自动踢出故障节点
    - 存储进程的执行状态

## Etcd性能

- Raft
    - 网络IO节点之间的RTT/带宽
    - WAL受到磁盘IO写入延迟
- Storage
    - 磁盘IOfdatasync延迟
    - 索引层锁的block
    - boltdb Tx的锁
    - boltdb本身的性能
- 其他
    - 内核参数
    - grpc api层延迟

### 调优

- Server性能优化
    - 硬件
        - 升级CPU/MEM
        - 选取高性能SSD
        - 网络带宽
        - 独立部署
    - 软件
        - 内存索引层
        - lease规模使用
        - 后端boltdb使用优化
- client性能优化
    - put时避免大的value，精简再精简 ex: k8s crd使用
    - 避免创建频繁变化的key/value ex: k8s node数据上传
    - 避免创建大量lease，尽量选择复用 ex: k8s event数据管理

# 调度和资源管理

## 调度过程

把Pod放到合适的Node上

- 满足Pod资源要求
- 满足Pod的特殊关系要求
- 满足Node限制条件要求
- 做到集群资源合理利用

## 基础调度能力

### 资源调度 - 满足Pod资源要求

- Resources: CPU/Memory/Storage/GPU/FGPA
- QoS(Quality of Service)
    - Guaranteed - 敏感型、需要保障业务
    - Burstable - 次敏感型、需要弹性业务
    - Bestffort - 可容忍型业务
- Resource Quota
    - per namespace
    - Scope:
        - Terminating/NotTerminating
        - BestEffort/NotBestEffort
        - PriorityClass

```yaml
# Resources
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: demo-pod
spec:
  containers:
  - image: nginx
    name: demo-container
    resources:
      requests:
        cpu: 2
        memory: 1Gi
      limits:
        cpu: 2
        memory: 1Gi

# Quota
apiversion: v1
kind: ResourceQuota
metadata:
  name: demo-quota
  namespace: default
spec:
  hard:
    cpu: 1000
    memory: 200Gi
    pods: 100
  scopeSelector:
    mathExpressions:
    - operator: Exists
      scopeName: NotBestEffort
```

### 关系调度 - 满足Pod/Node的特殊关系、条件要求

- PodAffinity/PodAntiAffinity: Pod与Pod间关系
    - requiredDuringSchedulingIgnoredDuringExecution
    - preferredDuringSchedulingIgnoreDuringExecution
- NodeSelector/NodeAffinity: Pod决定适合的Node
    - NodeSelector: Map[string]string
    - NodeAffinity
        - requiredDuringSchedulingIgnoredDuringExecution
        - preferredDuringSchedulingIgnoreDuringExecution
- Taint/Tolerations: 限制调度到某些Node
    - Node Taints
        - Effect
            - NoSchedule - 只禁止新的Pods调度上来
            - PreferNoSchedule - 尽量不调度到这台
            - NoExecute - 会evict没有对应的toleration的Pods，并且不会调度新的上来
    - Pod Tolerations
        - Effect
            - 取值与Taints的Effect一致
        - Operator
            - Exists/Equal

## 高级调度能力

### 优先级调度和抢占

- Priority
- Preemption

### 资源不够时如何调度

- 先到先得策略（FIFO） - 简单、相对公平、上手快
- 优先级策略（Priority） - 符合日常公司业务特点
    - PodPriority & Preemption
        - v1.14 - stable
        - default is ON

### 优先级调度配置

1. 创建PriorityClass

    ```yaml
    apiVersion: scheduling.k8s.io/v1
    kind: PriorityClass
    metadata:
      name: high
    value: 10000
    globalDefault: false
    ```

2. 为实例配置山不同的priorityClassName

    ```yaml
    spec:
      priorityClassName: high
    ```

3. 默认优先级说明
    - 内置默认优先级 DefaultPriorityWhenNoDefaultClassExists = 0
    - 用户可配置的最大优先级限制 HighestUserDefinablePriority = 1000000000
    - 系统级别优先级 SystemCriticalPriority = 2000000000
    - 内置系统级别优先级
        - system-cluster-critical
        - system-node-critical

### 优先级抢占策略

PDB(PodDisruptionBudget)

![kubernetes%208cb51f8e78ef4e5fbef90eaac39d554b/Screen_Shot_2020-12-07_at_5.06.18_PM.png](kubernetes%208cb51f8e78ef4e5fbef90eaac39d554b/Screen_Shot_2020-12-07_at_5.06.18_PM.png)

## 调度器架构和具体算法

[](https://www.bilibili.com/video/BV12J411q7k4)

### 调度流程

![kubernetes%208cb51f8e78ef4e5fbef90eaac39d554b/Untitled.png](kubernetes%208cb51f8e78ef4e5fbef90eaac39d554b/Untitled.png)

### 调度算法

### 配置调度器

# GPU和Device Plugin

 pass

# API编程

## API编程范式 - CRD(Custom resources definition)

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: foos.samplecontroller.k8s.io
spec:
  group: samplecontroller.k8s.io
  version: v1alpha1
  names:
    kind: Foo
    plural: foos
  scope: Namespaced

apiVersion: samplecontroller.k8s.io/v1alpha1
kind: Foo
metadata:
  name: ex-foo
spec:
  deploymentName: ex-foo
  replicas: 1
```

## API编程利器 - Operator and Operator Framework

### 基本概念

- CRD: Custom Resource Definition，允许用户自定义kubernetes资源
- CR: Custom Resource，CRD的具体实例
- webhook: webhook关联在api server上，是一种HTTP回调，一个基于web应用实现的webhook会在特定事件发生时把消息发送给特定的URL
    - mutating webhook - 变更传入对象
    - validating webhook - 传入对象校验
- 工作队列: controller核心组件，controller会监控集群内关注资源对象的变化，并把相关事件（动作和key）存储于工作队列中
- controller: 检测集群状态变化，并据此作出相应处理的控制循环
- operator: 描述、部署和管理kubernetes 应用的一套机制，operator = CRD + webhook + controller

![kubernetes%208cb51f8e78ef4e5fbef90eaac39d554b/Untitled%201.png](kubernetes%208cb51f8e78ef4e5fbef90eaac39d554b/Untitled%201.png)

### Operator framework

- kubebuilder

    [kubernetes-sigs/kubebuilder](https://github.com/kubernetes-sigs/kubebuilder)

- operrator-sdk

    [operator-framework/operator-sdk](https://github.com/operator-framework/operator-sdk)

# 安全

## 访问控制

### API请求访问控制

- Authentication: 用户是否为能够访问集群的合法用户
- Authorization: 用户是否有权限进行请求中的操作
- Admission Control: 请求是否安全合规

### 认证

- Basic
- X509证书
    - 认证机构CA
        - 公钥 - /etc/kubernetes/pki/ca.crt
        - 私钥 - /etc/kubernetes/pki/ca.key
    - 集群组件间通讯用证书都是由集群根CA签发
    - 两个身份凭证相关的重要字段
        - Common Name(CN): apiserver在认证过程中将其作为用户(user)
        - Organization(O): apiserver在认证过程中将其作为组(group)

    [组件对应的自身证书](https://www.notion.so/19c6ed9f4ec44f9f92ba5e4ded0f930d)

- Bearer Tokens(JSON Web Tokens)
    - Service Account
    - OpenID Connect
    - Webhooks

### RBAC

- 策略包含主体（subject）、动作（verb）、资源（resource）、命名空间（namespace）
    - User A can create pods in namespace B
- 默认拒绝所有访问: deny all
- Role & RoleBinding

    ```yaml
    kind: Role
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      namespace: default
      name: test-pod
    rules:
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["get", "watch", "list"]

    ---
    kind: RoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: test-pod
      namespace: default
    subjects:
    - kind: User
      name: test
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: Role
      name: test-pod
      apiGroup: rbac.authorization.k8s.io

    # RoleBinding 可以绑定ClusterRole 但只能在对应的指定namespace下生效
    ```

- ClusterRole & ClusterRoleBinding

    ```yaml
    kind: ClusterRole
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: test-configmap
    rules:
    - apiGroups: [""]
      resources: ["configmaps"]
      verbs: ["get", "watch", "list"]

    ---
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: test-global
    subjects:
    - kind: Group
      name: test
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: ClusterRole
      name: test-configmap
      apiGroup: rbac.authorization.k8s.io

    ---
    # 用户类型
    # www命名空间里所有的service account
    subjects:
    - kind: Group
      name: system:serviceaccounts:www
      apiGroup: rbac.authorization.k8s.io

    # 所有service account
    subjects:
    - kind: Group
      name: system:serviceaccounts
      apiGroup: rbac.authorization.k8s.io

    # 所有用户
    subjects:
    - kind: Group
      name: system:authenticated   # 需要验证的
      apiGroup: rbac.authorization.k8s.io
    - kind: Group
      name: system:unauthenticated # 不需要验证的
      apiGroup: rbac.authorization.k8s.io
    ```

### Security Context - Runtime安全策略

- 遵循权限最小化原则
- 在Pod或Container维度设置Security Context
- 开启下列admission controllers
    - ImagePoliocyWebhook - 支持对接外部的webhook对部署镜像进行校验
    - AlwaysPullImages - 在多租环境下防止部署镜像被恶意篡改

[Untitled](https://www.notion.so/ff280eec0315437eb846343574ce05b5)