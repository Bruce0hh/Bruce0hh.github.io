# Kubernetes-Deamon

## Deamon Pod特征

> DaemonSet的主要作用，是让你在Kubernetes集群里，运行一个Daemon Pod。

1.  这个Pod运行在Kubernetes集群里的每一个节点上；
2.  每个节点上都有一个这样的Pod实例；
3.  当有新的节点加入Kubernetes集群后，该Pod会自动地在新节点上被创建出来；而当旧节点被删除后，它上面的Pod也会相应地被回收掉。

DaemonSet 是如何保证每个Node上有且只有一个被管理的Pod呢？显然，可以通过Labels（标签）去检查Pod。而检查的结果，可能就有三种情况：

1. 没有这种Pod，那么就意味着要在这个Pod上创建一个Pod；
2. 有这种Pod，但是数量大于1，那就说明要把多余的Pod从这个Node上删除掉；
3. 正好只有一个这种Pod，那说明这个节点是正常的。

其中，删除多余的Pod只要调用Kubernetes API就可以了。那么在指定的Node上创建新Pod，需要用nodeAffinity和toleration字段来进行调度。

## Tolerations（容忍）

- DaemonSet会给Pod自动加上一个与调度相关的字段，叫做tolerations。这个字段意味着这个Pod，会容忍（toleration）某些Node的污点（Taint）。
- Toleration的含义是：“容忍”所有被标记为 `unschedulable`污点的Node；“容忍”的效果是**允许调度**。
- 正常情况下，被标记了unschedulable“污点”的Node，是不会有任何Pod被调度上去的。可是，DaemonSet自动地给被管理的Pod加上了这个特殊的Toleration，就使得这些Pod可以忽略这个限制，继而保证每个节点上都会被调度一个Pod。

DaemonSet其实是一个非常简单的控制器。在它的控制循环中，只需要遍历所有节点，然后根据节点上是否有被管理Pod情况，来决定是否要创建或者删除一个Pod。只不过，在创建每个Pod的时候，DaemonSet会自动给这个Pod加上一个nodeAffinity，从而保证这个Pod只会在指定节点上启动。同时，它还会自动给这个Pod加上一个Toleration,从而忽略节点的unschedulable“污点”。

DaemonSet使用ControllerRevision，来保存和管理自己对应的“版本”。这种面向API对象的设计思路，大大简化了控制器本身的逻辑，也正是Kubernetes项目声明式API的优势所在。在Kubernetes项目里，ControllerRevision其实是一个通用的版本管理对象。这样，Kubernetes项目就巧妙地避免了每种控制器都要维护一套冗余的代码和逻辑的问题。