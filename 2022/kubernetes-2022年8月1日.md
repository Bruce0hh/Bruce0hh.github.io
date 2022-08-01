# Kubernetes-Job/CronJob

## Job 基本概念

```yaml
apiVersion: batch/v1
kind: Job
metadata:
	name: pi
spec:
	template:
		spec:
			containers:
			- name: pi
			  image: resouer/ubuntu-bc
			  command: ["sh", "-c", "eche 'scale=10000;4*a(1)' | bc -l"]
		 restartPolicy: Never
     backoffLimit: 4
```



这个Pod创建后，它的Pod模版，被自动加上了一个`controller-uid=<一个随机字符串>`这样的Label。而这个Job对象本身，则被自动加上了这个Label对应的Selector，从而保证了Job与它所管理的Pod之间的关系。但是这种自动生成的Label对用户来说并不友好，所以不太适合用到Deployment等长作业编排对象上。



离线计算的Pod永远都不应该被重启，否则它们会再被重新计算一遍——这也是我们需要在Pod模版中定义`restartPolicy=Never`的原因。`backoffLimit`字段保证了当离线作业失败时，创建Pod的重试次数为4。需要注意的是，Job Controller重新创建Pod的间隔是呈指数增加的。



## 并行作业

在Job对象中，负责并行控制的参数有两个：

1. `spec.parallelism`，定义的是一个Job在任意时间最多可以启动多少个Pod同时运行；
2. `spec.completions`，定义的是Job至少要完成的Pod数目，即Job的最小完成数。

### Job Controller 工作原理

1. Job Controller控制的对象，直接就是Pod。
2. JC在控制循环中进行的调谐（Reconcile）操作，是根据实际在Running状态Pod的数目、已经成功退出的Pod数目，以及parallelism、completions参数的值共同计算出在这个周期内，应该创建或者删除的Pod数目，然后调用Kubernetes API来执行这个操作。
3. JC实际上控制了，作业执行的并行度，以及总共需要完成的任务数这两个重要参数。而在实际使用时，需要根据作业的特性，来决定并行度（parallelism）和任务数（completions）的合理取值。

## 常用Job对象的方法

1. 外部管理器+Job模版
2. 拥有固定任务数目的并行Job
3. 指定并行度（parallelism），但不设置固定的任务数（completions）

## CronJob

本身就是一个Job对象的控制器。相较于Job，多了一个`schedule`字段，通过cron表达式创建Job对象。

`spec.concurrencyPolicy`：对于定时任务，当一个Job未执行完，新的Job就产生时的调度策略

1. `concurrencyPolicy=Allow`：这意味着这些多个Job可以同时存在。
2. `concurrencyPolicy=Forbid`：不会创建新的Job，该创建周期被跳过。
3. `concurrencyPolicy=Replace`：新的Job会替换旧的、未执行完的Job。

