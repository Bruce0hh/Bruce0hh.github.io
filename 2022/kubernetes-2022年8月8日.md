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
   - Validation则负责验证这个对象里的各个字段是否合法。

   


## 自定义控制器