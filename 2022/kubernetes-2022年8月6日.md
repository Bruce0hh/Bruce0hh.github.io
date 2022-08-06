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

> 声明式API，才是Kubernetes项目编排能力“赖以生存”的核心所在。



### Istio

> Istio最根本的组件，是运行在每一个应用Pod里的Envoy容器里。

<img src="https://cdn.jsdelivr.net/gh/Bruce0hh/pic-bed/20220807013955.png" style="zoom: 50%;" />



## API对象

> 在Kubernetes项目中，一个API对象在Etcd里的完整资源路径，是由：Group（API组）、Version（API版本）和Resource（API资源类型）三个部分组成的。

### API解析流程
1. Kubernetes会匹配API对象的组。
2. Kubernetes会进一步匹配到API对象的版本号。
3. Kubernetes会匹配API对象的资源类型。

### API对象创建流程


## 自定义控制器