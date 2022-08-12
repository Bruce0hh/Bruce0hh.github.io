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
2. Informer使用client跟APIServer建立了连接。实际上是使用Informer中Reflector的`ListAndWatch`方法，来获取监听Network对象实例的变化。
3. ListAndWatch方法的含义是：首先，通过APIServer的LIST API获取所有最新版本的API对象；然后通过WATCH API来监听所有这些API对象的变化。
4. 当APIServer端有新的Network实例被创建、删除或更新，Reflector都会收到事件通知，然后该事件以及对应的API对象，就会被放进这个一个`Delta FIFO Queue`中。
5. Informer会不断地从DFQ中读取Pop增量。没拿到一个增量，Informer就会判断这个增量的事件类型，然后通过`Indexer`的库把增量中的API对象保存在本地缓存（`Local Store`）中。

### 编写自定义控制器的定义

```go
func NewController(
  kubeclientset kubernetes.Interface,
  networkclientset clientset.Interface,
  networkInformer informers.NetworkInformer) *Controller {
  ...
  controller := &Controller{
    kubeclientset:    kubeclientset,
    networkclientset: networkclientset,
    networksLister:   networkInformer.Lister(),
    networksSynced:   networkInformer.Informer().HasSynced,
    workqueue:        workqueue.NewNamedRateLimitingQueue(...,  "Networks"),
    ...
  }
    networkInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: controller.enqueueNetwork,
    UpdateFunc: func(old, new interface{}) {
      oldNetwork := old.(*samplecrdv1.Network)
      newNetwork := new.(*samplecrdv1.Network)
      if oldNetwork.ResourceVersion == newNetwork.ResourceVersion {
        return
      }
      controller.enqueueNetwork(new)
    },
    DeleteFunc: controller.enqueueNetworkForDelete,
 return controller
}
```

1. 设置一个工作队列`Work Queue`，负责同步Infomer和控制循环（`Control Loop`）之间的数据。
2. 为Informer注册Handler（AddFunc、UpdateFunc和DeleteFunc），分别对应API对象的“添加”“更新”和“删除”事件。具体操作就是将这些事件对应的API对象加入到Work Queue。实际上入队的只是API对象的Key。
3. 通过监听事件变化，Informer就可以实时地更新本地缓存，并且调用事件的EventHandler了。Informer维护的本地缓存，都会使用最近一次LIST API返回的结果强制更新，从而保证缓存的有效性。这个缓存强制更新的操作就叫作：`resync`。
4. 最后是控制循环：首先，等待Informer完成一次本地缓存的数据同步操作；然后，直接通过goroutine启动一个或并发多个“无限循环”的任务。

### 编写控制器里的业务逻辑

1. 首先从工作队列中出队了一个成员，也就是一个Key；
2. 然后，在syncHandler方法中，尝试从Informer维护的缓存中拿到了它所对应的Network对象。
3. 如果控制循环从缓存中拿不到这个对象，说明这个Network对象的Key是通过前面的“删除”事件添加到工作队列，应该使用**Neutron的API**，把这个Key对应的Neutron网络从真实的集群里删除掉。
4. Informer将“期望状态”的API对象缓存到本地；又可以通过Neutron API来查询“实际状态”的API对象。
5. 有两个状态的对比，就可以决定业务逻辑了。



