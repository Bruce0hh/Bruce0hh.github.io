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



### Istio

- Istio最根本的组件，是运行在每一个应用Pod里的Envoy容器里。
- 而Istio项目，则把这个Envoy代理服务以sidecar容器的方式，运行在了每一个被治理的应用Pod中。
- Pod里所有的容器都共享同一个Network Namespace。所以，Envoy容器就能够通过配置Pod里的iptables规则，把整个Pod的进出流量接管下来。
- 这时候，Istio的控制层（Control Plane）里的Pilot组件，就能够通过调用每个Envoy容器的API，对这个Envoy代理进行配置，从而实现微服务治理。

![](https://cdn.jsdelivr.net/gh/Bruce0hh/pic-bed/20220807013955.png)

假设这个架构图左边的Pod是已经在运行的应用，而右边的Pod则是我们刚刚上线应用的新版本。此时，Pilot通过调节这两个Pod里的Envoy容器的配置，从而将90%的流量分配给旧版本的应用，将10%的流量分配给新版本应用，并且，还可以在后续的过程中随时调整。

这样，一个典型的“灰度发布”场景就完成了。比如Istio可以调节这个流量从90%-10%，改到80%-20%，再到50%-50%，最后到0%-100%，就完成了这个灰度发布的过程。

## Dynamic Admission Control

>  Istio项目使用的，是Kubernetes中的一个功能——`Dynamic Admission Control`。通过这个功能，Istio在Envoy容器的部署或Envoy代理的部署，对用户和应用来说都是无感的。

在Kubernetes项目中，当一个Pod或者任何一个API对象被提交给APIServer之后，总有一些“初始化”性质的工作需要在它们被Kubernetes项目正式处理之前进行。



## API对象

> 在Kubernetes项目中，一个API对象在Etcd里的完整资源路径，是由：Group（API组）、Version（API版本）和Resource（API资源类型）三个部分组成的。

### API解析流程
1. Kubernetes会匹配API对象的组。
2. Kubernetes会进一步匹配到API对象的版本号。
3. Kubernetes会匹配API对象的资源类型。

### API对象创建流程


## 自定义控制器