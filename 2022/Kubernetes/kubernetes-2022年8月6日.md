[TOC]

## 声明式API

### docker

``` shell
$ docker service create --name nginx --replicas 2 nginx
$ docker service update --image nginx:latest nginx
```

- 第一条create命令创建了容器，而第二条update命令则把它们“滚动更新”成了一个新的镜像。

- 这种使用方式，我们称为**命令式命令行操作**。

### kubernetes

```shell
$ kubectl create -f nginx.yaml
$ kubectl replace -f nginx.yaml
```

- 对于上面这种先kubectl create，再replace的操作，我们称为**命令式配置文件操作**。
- 实际上和上述的docker操作本质是一样的，通过create创建，通过replace“滚动更新”。

```shell
$ kubectl apply -f nginx.yaml
```

- kubectl replace的执行过程，是使用新的yaml文件中的API对象，**替换原有的API对象**；而kubectl apply，则是执行了一个**对原有对象的PATCH操作**。这就是**声明式API**。
- 响应命令式请求：一次只能处理一个写请求。
- 声明式请求：一次能处理多个写操作，并且具备Merge能力。



### 声明式API特点

1. 首先，所谓“声明式”，指的就是我只需要提交一个定义好的API对象来“声明”，我所期望的状态是什么样子。
2. 其次，“声明式API”允许有多个API写端，以PATCH的方式对API对象进行修改，而无需关心原始YAML文件的内容。
3. 最后，也是最重要的，有了上述两个能力，Kubernetes项目才可以基于对于API对象的增删改查，在完全无需外界干预的情况下，完成对“实际状态”和“期望状态”的调谐（Reconcile）过程。

**声明式API，才是Kubernetes项目编排能力“赖以生存”的核心所在。**



## Istio

- Istio最根本的组件，是运行在每一个应用Pod里的Envoy容器里。
- 而Istio项目，则把这个Envoy代理服务以sidecar容器的方式，运行在了每一个被治理的应用Pod中。
- Pod里所有的容器都共享同一个Network Namespace。所以，Envoy容器就能够通过配置Pod里的iptables规则，把整个Pod的进出流量接管下来。
- 这时候，Istio的控制层（Control Plane）里的Pilot组件，就能够通过调用每个Envoy容器的API，对这个Envoy代理进行配置，从而实现微服务治理。

![](https://gcore.jsdelivr.net/gh/Bruce0hh/Bruce0hh.github.io/pic-bed/Istio-Envoy.png)

假设这个架构图左边的Pod是已经在运行的应用，而右边的Pod则是我们刚刚上线应用的新版本。此时，Pilot通过调节这两个Pod里的Envoy容器的配置，从而将90%的流量分配给旧版本的应用，将10%的流量分配给新版本应用，并且，还可以在后续的过程中随时调整。

这样，一个典型的“灰度发布”场景就完成了。比如Istio可以调节这个流量从90%-10%，改到80%-20%，再到50%-50%，最后到0%-100%，就完成了这个灰度发布的过程。

### Dynamic Admission Control

>  Istio项目使用的，是Kubernetes中的一个功能——`Dynamic Admission Control`。通过这个功能，Istio在Envoy容器的部署或Envoy代理的部署，对用户和应用来说都是无感的。

- 在Kubernetes项目中，当一个Pod或者任何一个API对象被提交给APIServer之后，总有一些“初始化”性质的工作需要在它们被Kubernetes项目正式处理之前进行。

- 而这个“初始化”操作的实现，借助的是一个叫作Admission的功能。它其实是Kubernetes项目里一组被称为Admission Controller的代码，可以选择性地被编译进APIServer中，在API对象创建之后会被立刻调用到。
- 如果你现在想要添加一些自己的规则到Admission Controller，就会比较困难。因为，这要求重新编译并重启APIServer。所以Kubernetes项目为用户提供了一种“热插拔”式的Admission机制，就是DAC，也叫做：Initializer。

### Initialize

> 当Pod YAML文件被提交给Kubernetes之后，Istio项目要做的就是，在它对应的API对象里自动加上Envoy容器的配置。

Istio要做的，就是编写一个用来为Pod“自动注入”Envoy容器的Initializer。

1. 首先，Istio会将这个Envoy容器本身的定义，以ConfigMap的方式保存在Kubernetes当中。
   - Initializer要做的工作，就是把ConfigMap中与Envoy相关的字段，自动添加到用户提交的Pod的API对象里。在Initializer更新用户的Pod对象时候，必须使用PATCH API来完成。而这种PATCH API，正是声明式API最主要的能力。
2. 接下来，Istio将一个编写好的Initializer，作为一个Pod部署在Kubernetes中。
   - Initializer的控制器实际就是一个死循环，不断地获取用户新创建的Pod。
   - 如果这个Pod里面已经添加过Envoy容器，那么就放过这个Pod，进入下一个周期。
   - 如果没有添加过Envoy容器，它就要进行Initialize操作了。
3. Kubernetes的API库，为我们提供了一个方法，使得我们可以直接使用新旧两个Pod对象，生成一个`TwoWayMergePatch`。有了这个TwoWayMergePatch之后，Initializer的代码就可以使用这个patch的数据，**调用Kubernetes的client，发起一个PATCH请求**。
4. 当你在Initializer里完成了要做的操作后，一定要记得将`metadata.initializers.pending`标志清除。
   - 因为每一个新创建的Pod，都会自动携带了`metadata.initializers.pending`的Metadata信息。
   - Metadata信息，正是Initializer的控制器判断这个Pod有没有执行过自己所负责的初始化操作的重要依据。

**Istio项目的核心，就是由无数个运行在应用Pod中的Envoy容器组成的服务代理网格。这也正是Service Mesh的含义。**

