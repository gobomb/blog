---
layout: post
title: "kubernetes 亲和性调度"
excerpt: "除了让 kubernetes 集群调度器自动为 pod 资源选择某个节点（默认调度考虑的是资源足够，并且 load 尽量平均），有些情况我们希望能更多地控制 pod 应该如何调度。"
categories: blog
tags: [kubernetes, scheduler]
comments: true
share: true
---

## kubernetes pod 调度简介

除了让 [kubernetes 集群调度器](http://cizixs.com/2017/03/10/kubernetes-intro-scheduler)自动为 pod 资源选择某个节点（默认调度考虑的是资源足够，并且 load 尽量平均），有些情况我们希望能更多地控制 pod 应该如何调度。比如，集群中有些机器的配置更好（ SSD，更好的内存等），我们希望比较核心的服务（比如说数据库）运行在上面；或者某两个服务的网络传输很频繁，我们希望它们最好在同一台机器上，或者同一个机房。

这种调度在 kubernetes 中分为两类：node affinity 和 pod affinity。

## 选择 node

kubernetes 中有很多对 label 的使用，node 就是其中一例。label 可以让用户非常灵活地管理集群中的资源，service 选择 pod 就用到了 label。这篇文章介绍到的调度也是如此，可以根据节点的各种不同的特性添加 label，然后在调度的时候选择特定 label 的节点。

在使用这种方法之前，需要先给 node 加上 label，通过 `kubectl` 非常容易做：

```
kubectl label nodes <node-name> <label-key>=<label-value>
```

列出 node 的时候指定 `--show-labels` 参数就能查看 node 都添加了哪些 label：

```
kubectl get nodes --show-labels
```

node 有了 label ，在调度的时候就可以用到这些信息，用法也很简单，在 pod 的 spec 字段下面加上 `nodeSelector`，它里面保存的是多个键值对，表示节点上有对应的 label，并且值也匹配。还是举个例子看得明白，比如原来简单的 nginx pod：

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
```

添加上 node 选择信息，就变成了下面这样：

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```

这个例子就是告诉 kubernetes 调度的时候把 pod 放到有 SSD 磁盘的机器上。

除了自己定义的 label 之外，kubernetes 还会自动给集群中的节点添加一些 label，比如：

- `kubernetes.io/hostname`：节点的 hostname 名称
- `beta.kubernetes.io/os`： 节点安装的操作系统
- `beta.kubernetes.io/arch`：节点的架构类型
- ……

不同版本添加的 label 会有不同，这些 label 和手动添加的没有区别，可以通过 `--show-labels` 查看，也能够用在 `nodeSelector` 中。

**NOTE**：nodeSelector 的方式比较简单直观，但是不够灵活。以后，它会被我们下面讲到的 `Node Affinity` 替代，请关注每个版本的 release note。

## Node Affinity

`Affinity` 翻译成中文是“亲和性”，它对应的是 `Anti-Affinity`，我们翻译成“互斥”。这两个词比较形象，可以把 pod 选择 node 的过程类比成磁铁的吸引和互斥，不同的是除了简单的正负极之外，pod 和 node 的吸引和互斥是可以灵活配置的。

kubernetes 1.2 版本开始引入这个概念，目前（1.6版本）处于 beta 阶段，相信后面会变成核心的功能。这种方法比 nodeSelector 复杂，但是也更灵活，提供了更精细的调度控制。它的优点包括：

- 匹配有更多的逻辑组合，不只是字符的完全相等
- 调度分成软策略（soft）和硬策略（hard），在软策略的情况下，如果没有满足调度条件的节点，pod 会忽略这条规则，继续完成调度过程

目前有两种主要的 node affinity： `requiredDuringSchedulingIgnoredDuringExecution` 和 `preferredDuringSchedulingIgnoredDuringExecution`。前者表示 pod 必须部署到满足条件的节点上，如果没有满足条件的节点，就不断重试；后者表示优先部署在满足条件的节点上，如果没有满足条件的节点，就忽略这些条件，按照正常逻辑部署。

`IgnoredDuringExecution` 正如名字所说，pod 部署之后运行的时候，如果节点标签发生了变化，不再满足 pod 指定的条件，pod 也会继续运行。与之对应的是 `requiredDuringSchedulingRequiredDuringExecution`，如果运行的 pod 所在节点不再满足条件，kubernetes 会把 pod 从节点中删除，重新选择符合要求的节点。

**吐槽**：命名果然是软件技术的难点之一，kubernetes 开发者命名采取的技巧是——简单粗暴：直接把需求用语言表达出来，这也导致这个名字看起来非常奇怪。

软策略和硬策略的区分是有用处的，硬策略适用于 pod 必须运行在某种节点，否则会出现问题的情况，比如集群中节点的架构不同，而运行的服务必须依赖某种架构提供的功能；软策略不同，它适用于满不满足条件都能工作，但是满足条件更好的情况，比如服务最好运行在某个区域，减少网络传输等。这种区分是用户的具体需求决定的，并没有绝对的技术依赖。

拿官方文档的例子来说明：

```
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: gcr.io/google_containers/pause:2.0
```

这个 pod 同时定义了 `requiredDuringSchedulingIgnoredDuringExecution` 和 `preferredDuringSchedulingIgnoredDuringExecution` 两种 nodeAffinity。第一个**要求** pod 运行在特定 AZ 的节点上，第二个**希望**节点最好有对应的 `another-node-label-key:another-node-label-value` 标签。

这里的匹配逻辑是 label 的值在某个列表中，可选的操作符有：

- `In`：label 的值在某个列表中
- `NotIn`：label 的值不在某个列表中
- `Exists`：某个 label 存在
- `DoesNotExist`: 某个 label 不存在
- `Gt`：label 的值大于某个值（字符串比较）
- `Lt`：label 的值小于某个值（字符串比较）

并没有 node anti-affinity 这种东西，因为 `Notin` 和 `DoesNotExist` 能提供类似的功能。

如果`nodeAffinity` 中 `nodeSelectorTerms` 有多个选项，如果节点满足任何一个条件就可以；如果 `matchExpressions` 有多个选项，则只有同时满足这些逻辑选项的节点才能运行 pod。

### Pod Affinity

通过上一部分内容的介绍，我们知道怎么在调度的时候让 pod 灵活地选择 node；但有些时候我们希望调度能够考虑 pod 之间的关系，而不只是 pod-node 的关系。pod affinity 是在 kubernetes1.4 版本引入的，目前在 1.6 版本也是 beta 功能。

为什么有这样的需求呢？举个例子，我们系统服务 A 和服务 B 尽量部署在同个主机、机房、城市，因为它们网络沟通比较多；再比如，我们系统数据服务 C 和数据服务 D 尽量分开，因为如果它们分配到一起，然后主机或者机房出了问题，会导致应用完全不可用，如果它们是分开的，应用虽然有影响，但还是可用的。

pod affinity 可以这样理解：调度的时候选择（或者不选择）这样的节点 N ，这些节点上已经运行了满足条件 X。条件 X 是一组 label 选择器，它必须指明作用的 namespace（也可以作用于所有的 namespace），因为 pod 是运行在某个 namespace 中的。

和 node affinity 相似，pod affinity 也有 `requiredDuringSchedulingIgnoredDuringExecution` 和 `preferredDuringSchedulingIgnoredDuringExecution`，意义也和之前一样。如果有使用亲和性，在 `affinity` 下面添加 `podAffinity` 字段，如果要使用互斥性，在 `affinity` 下面添加 `podAntiAffinity` 字段。

下面是一个例子：

```
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: failure-domain.beta.kubernetes.io/zone
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: kubernetes.io/hostname
  containers:
  - name: with-pod-affinity
    image: gcr.io/google_containers/pause:2.0
```

这个例子中，pod 需要调度到某个 zone（通过 `failure-domain.beta.kubernetes.io/zone` 指定），这个 zone 至少有一个节点上运行了这样的 pod：这个 pod 有 `security:S1` label。互斥性保证节点最好不要调度到这样的节点，这个节点上运行了某个 pod，而且这个 pod 有 `security:S2` label。

在 `labelSelector` 和 `topologyKey` 同级，还可以定义 `namespaces` 列表，表示匹配哪些 namespace 里面的 pod，默认情况下，会匹配定义的 pod 所在的 namespace；如果定义了这个字段，但是它的值为空，则匹配所有的 namespaces。

## 参考资料

- [Kubernetes Doc: Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/)
- [Node affinity and NodeSelector 设计文档](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/nodeaffinity.md)
- [pod affinity and anti-affinity 设计文档](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/podaffinity.md)
