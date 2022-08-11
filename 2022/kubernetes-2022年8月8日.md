[TOC]

## API对象

> 在Kubernetes项目中，一个API对象在Etcd里的完整资源路径，是由：Group（API组）、Version（API版本）和Resource（API资源类型）三个部分组成的。

### API解析流程

**Kubernetes里API对象的组织方式，其实是层层递进的。**

1. Kubernetes会匹配API对象的组。
2. Kubernetes会进一步匹配到API对象的版本号。
3. Kubernetes会匹配API对象的资源类型。

### API对象创建流程（以CronJob为例）

1. 当我们发起创建CronJob的POST请求之后，我们编写的YAML的信息就交给了`APIServer`。

   - APIServer就是过滤这个请求，并完成一些前置性的工作，比如授权、超时处理、审计等。

2. 请求就会进入`MUX`和`Routes`流程。

   - MUX和Routes是APIServer完成URL和Handler绑定的场所。APIServer根据API解析的流程完成匹配。

3. APIServer根据CronJob类型定义和YAML文件的字段，创建一个CronJob对象。

   - APIServer会进行一个Convert的工作，就是将YAML文件转换成一个`Super Version`对象，它是该API资源类型所有版本的字段全集。这样不同版本的YAML文件，就都可以用Super Version对象统一处理了。

4. APIServer会先后进行`Admission()`和`Validation()`操作。

   - Admission即包括Admission Controller和Initializer。
   - Validation则负责验证这个对象里的各个字段是否合法。这个被验证过的API对象，都保存在了APIServer里一个叫做`Registry`的数据结构中。

5. 最后，APIServer会把验证过的API对象转换成用户最初提交的版本，进行序列化操作，并调用Etcd的API把它保存起来。

   

## 自定义资源对象

[Custom Resource Definition (CRD)](https://github.com/resouer/k8s-controller-custom-resource)

## 自定义控制器（以Network对象的控制器为例）

> 基于声明式API的业务功能实现，往往需要通过控制器模式来“监视”API对象的变化，然后一次来决定实际要执行的具体工作。

### 编写main函数

```go
func main() {
  ...
  
  cfg, err := clientcmd.BuildConfigFromFlags(masterURL, kubeconfig)
  ...
  kubeClient, err := kubernetes.NewForConfig(cfg)
  ...
  networkClient, err := clientset.NewForConfig(cfg)
  ...
  
  networkInformerFactory := informers.NewSharedInformerFactory(networkClient, ...)
  
  controller := NewController(kubeClient, networkClient,
  networkInformerFactory.Samplecrd().V1().Networks())
  
  go networkInformerFactory.Start(stopCh)
 
  if err = controller.Run(2, stopCh); err != nil {
    glog.Fatalf("Error running controller: %s", err.Error())
  }
}
```



main函数主要通过三步完成了初始化并启动一个自定义控制器的工作。

1. main函数根据用户提供的Master配置（APIServer的地址端口和kubeconfig的路径），创建一个Kubernetes的client和Network对象的client。

   Kubernetes 里所有的Pod都会以Volume的方式自动挂载Kubernetes的默认ServiceAccount。所以，这个控制器就会直接使用默认ServiceAccount数据卷里的授权信息，来访问APIServer。

2. main函数为Network对象创建一个叫做`InformerFactory`的工厂，并使用它生成一个Network对象的Informer，传递给控制器。

3. 启动上述的Informer，然后执行`controller.Run`，启动自定义控制器。

### 自定义控制器的工作原理

![自定义控制器工作流程](https://cdn.jsdelivr.net/gh/Bruce0hh/Bruce0hh.github.io/pic-bed/20220812010356.png)

1. 控制器要从Kubernetes的APIServer里获取Network对象。
   - 这个操作是依靠一个叫做Informer的代码库完成的。Informer与API对象是一一对应的，所以传递给自定义控制器的，是一个Network对象的Informer。
   - 在创建这个Informer工厂时，需要给它传递一个networkClient。Informer通过这个跟APIServer建立了连接。
   - 真正负责维护连接的，是Informer所使用的Refector包。使用其中的`ListAndWatch`的方法，来“获取”并“监听”这些Network对象实例的变化。

### 编写自定义控制器的定义



### 编写控制器里的业务逻辑