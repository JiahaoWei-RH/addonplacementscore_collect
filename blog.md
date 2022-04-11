
# Prototype of extensible scheduling using resources usage.
## Background
We already support extensible placement scheduling, which allows use of addonplacementscore to select clusters, but we lack an addonplacementscore that contains cluster resource usage information.

I developed a new addon based on addon-framework, which can collect resource usage information on the cluster, and generate an addonplacementscore under the cluster namespace of the hub, and then the placement uses addonplacementscore to select the cluster.

## Addonplacementscore 中的内容
首先看一下Addonplacementscore中的内容。scores中可以使用K-V格式保存agent中收集到的数据。K是自己设置的值，之后会使用到这个K值。


```
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: AddOnPlacementScore
...
  scores:
  - name: "cpuAvailable"
    value: 66
  - name: "memAvailable"
    value: 55
```
在这个addon创建的Addonplacementscore中，都会有两个score。
- cpuAvailable： 体现了当前managedcluster cpu可用的情况，value越大，说明该cluster越空闲

- memAvailable：体现了当前managedcluster mem可用的情况，value越大，说明该cluster的空闲mem更大

> 通过修改此addon中的agent代码(此处贴一个link)，可以设置自己需要的K-V来供自己使用。

## How to collect the resource usage of the cluster
cpuAvailable和memAvailable 都代表的是可用量，也就是剩余可以用来分配的资源。当前k8s的api中，没有直接计算可用量的方法。所以这两个值都是通过计算得到。

通过`kubectl describe node` 可以得到node的两个重要属性：Capacity和Allocatable


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ce5e8c7affe847148af78618664c535e~tplv-k3u1fbpfcp-watermark.image?)

通过对[官网文档](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/)的学习，可以知道Allocatable is defined as the amount of compute resources that are available for pods.

如果cluster只存在一个node，那么使用node.Allocatable 减去 所有当前cluster中所有pod使用的资源，就是可用量。如果cluster中存在多个node，可以得到如下公式，mem同理。

`cluster_cpu_avaiable = sum(node.status.Allocatable.cpu) - sum(pod_cpu_usage)`
`cluster_mem_avaiable = sum(node.status.Allocatable.mem) - sum(pod_mem_usage)`

从下面的图片来说，剩余蓝色部分代表的就是计算出来的可用量。


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8dc8b2357b454b1c86d6211f292db505~tplv-k3u1fbpfcp-watermark.image?)

## How to map the real value to the 0-100 range of addonplacementscore

上面那一步的计算只是完成了第一步，因为计算得到的是关于cluster属性的真实数值，而score中存放的value是一个相对数字(下文统一称为score.value)，并且数值在[-100, 100]这个区间。

因为最终的目的是需要通过addonplacementscore，从从个cluster中选择出几个cpu或者mem更加空闲的cluster，所以在真实值映射到score区间时，必须保证一点。

- 真实值和score.value 必须成正比

Here I use the maximum + normalization algorithm.

```
if value >= max
    score = 100
else 
    score = value *100 / max
```

Why？
1. If there is no maximum value, then an infinite interval is mapped to 0-100, and the average mapping cannot be achieved
2. When the available amount is greater than this maximum value, the cluster can be considered healthy enough to deploy applications

There may be a better algorithm here, and ideas are welcome.
## case
1. 选择出来cpuAvailbale最大的
2. 选择出来memAvailbale

## Summary

## Reference
