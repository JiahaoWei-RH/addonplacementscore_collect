# Placement extensible scheudling best practices
## Background
Open Cluster Management (OCM) is a community-driven project that is focused on multicluster and multicloud scenarios for Kubernetes applications. In OCM, the multicluster scheduling capabilities are provided by placement. As we have talked in previous article [Using the Open Cluster Management Placement for Multicluster Scheduling](https://cloud.redhat.com/blog/using-the-open-cluster-management-placement-for-multicluster-scheduling), you can use placement to filter clusters by label or claim selector, also placement provides some default prioritizers, you can use these prioritizers to sort clusters and select the most suitable ones.
​
One of the default prioritizers are ResourceAllocatableCPU/ResourceAllocatableMemory. It provides the capability to sort clusters based on the resource allocatable CPU or memory of managed clusters. However, when considering the resource based scheduling, we find in some cases, the prioritizer needs extra data (more than the allocatable CPU or memory) to calculate the score of the managed cluster. For example, there is a requirement to schedule based on resource monitoring data from the cluster. For this reason, we need a more extensible way to support scheduling based on customized scores.

## What is Placement extensible scheudling?

To support scheduling based on customized scores, OCM placement introduce an API `AddOnPlacementScore`. Details of the API's defination refer to [types_addonplacementscore.go](https://github.com/open-cluster-management-io/api/blob/main/cluster/v1alpha1/types_addonplacementscore.go). An example of `AddOnPlacementScore` is as below.

```
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: AddOnPlacementScore
metadata:
  name: default
  namespace: cluster1
status:
  conditions:
  - lastTransitionTime: "2021-10-28T08:31:39Z"
    message: AddOnPlacementScore updated successfully
    reason: AddOnPlacementScoreUpdated
    status: "True"
    type: AddOnPlacementScoreUpdated
  validUntil: "2021-10-29T18:31:39Z"
  scores:
  - name: "cpuAvailable"
    value: 66
  - name: "memAvailable"
    value: 55
```

All the customized scores information are stored in `status`, as we don't expect users to update it. It contains below 3 part.

* `conditions`. Conditions contain the different condition statuses for this AddOnPlacementScore.
* `validUntil`. ValidUntil defines the valid time of the scores. After this time, the scores are considered to be invalid by placement. nil means never expire. The controller owning this resource should keep the scores up-to-date.
* `scores`. Scores contain a list of score name and value of this managed cluster.

In above example, the API contains a list of customized scores: cpuAvailable and memAvailable. The score will be used by the score provider, the end user and placement.
* As a score provider, a 3rd party controller could run on either hub or managed cluster, to maintain the lifecycle of `AddOnPlacementScore` and update score into it.
* As an end user, you need to know the resource name "default" and customized score name "cpuAvailable"and "memAvailable" , so you can specify the name in placement yaml to select clusters. For example:
  ```
  apiVersion: cluster.open-cluster-management.io/v1beta1
  kind: Placement
  metadata:
    name: placement
    namespace: ns1
  spec:
    numberOfClusters: 3
    prioritizerPolicy:
      mode: Exact
      configurations:
        - scoreCoordinate:
            type: AddOn
            addOn:
              resourceName: default
              scoreName: cpuAvailable
          weight: 1
  ```
* In placement, if the end user defines the scoreCoordinate type as AddOn, placement will get `AddOnPlacementScore` resource with name "default" in each cluster's namespace, read score "cpuAvailable" in the score list, and use that score to sort clusters.

You can refer to the [enhancements](https://github.com/open-cluster-management-io/enhancements/blob/main/enhancements/sig-architecture/32-extensiblescheduling/32-extensiblescheduling.md) to learn more details about the design. In the design, how to maintain the lifecycle (create/update/delete) of the `AddOnPlacementScore` CRs is not covered, as we expect the 3rd party controller itself to manage it. And in this blog, we will use an example to show you how to implement a 3rd part controller to update your own scores and extend the multiple clusters scheduling capability with your own scores.


## How to implement a 3rd part controller

In the OCM environment, you can use the [addon-freamwork](https://github.com/open-cluster-management-io/addon-framework) framework to quickly develop an addon plugin. Here we use it to develop an addon with a hub-agent architecture.

To develop a controller, you need to consider the following three points.

The following is a specific [example](https://github.com/JiahaoWei-RH/addonplacementscore_collect) to show how to develop it. The main function of this example is which can collect resource usage information on the cluster, and generate an addonplacementscore under the cluster namespace of the hub, and then the placement uses addonplacementscore to select the cluster.

### 1. Where to run the 3rd party controller

The 3rd part controller could run on either hub or managed cluster. Combined with User Stories, you should be able to distinguish whether the controller should be placed in a hub or a managed cluster.

In the example, the controller is deployed on the managed cluster via [deployment](https://github.com/JiahaoWei-RH/addon-framework/blob/main/examples/socre-collect/manifests/templates/deployment.yaml) .

![image](https://user-images.githubusercontent.com/56222648/165246983-903d2bf3-7d38-47d8-b3d9-ca4aca2b60bf.png)

### 2. How to maintain the AddOnPlacementScore CR lifecycle
- When should the score be created ?

  It can be created with the existence of a ManagedCluster, or on demand for the purpose of reducing objects on hub.

  In the example, the addon will recalculate the score every 60 seconds. When the score is calculated for the first time, if it is not in the hub, it will be created first.

- When should the score be updated ?
  We recommend that you set ```ValidUntil``` when updating the score, so that the placement controller can know if the score is still valid in case it failed to update for a long time.
  The score could be updated when your monitoring data changes, or at least you need to update it before it expires.
  In this example, in addition to updating the score every 60 seconds, the update will also be triggered when the node or pod resource in the managedcluster changes.

  ```go
  factory.New().WithInformersQueueKeyFunc(
     func(obj runtime.Object) string {
        key, _ := cache.MetaNamespaceKeyFunc(obj)
        return key
     }, addOnPlacementScoreInformer.Informer()).
     WithBareInformers(podInformer.Informer(), nodeInformer.Informer()).
     WithSync(c.sync).ResyncEvery(time.Second*60).ToController("score-agent-controller", recorder)
  ```

- When should the score be deleted?

### 3. How to calculate the score

The score must be in the range -100 to 100, you need to normalize the scores before update it into AddOnPlacementScore. We can divide the normalization of data into two cases according to the specific situation

- know the actual value of max
  In this case, it is easy to achieve smooth mapping by formula. Suppose the actual value is X, and X is in the interval [min, max], then``` score ＝ (100 + 100)/(max - min) * (x - min) - 100  ```

- don't know the actual value of max

  But in this case, we often need to set a maximum value ourselves, The reason is that:

  1. If there is no maximum value, then an infinite interval is mapped to 0-100, and the average mapping cannot be achieved
  2. When the available amount is greater than this maximum value, the cluster can be considered healthy enough to deploy applications.

  So relative to the first case, we need to add extra judgment:

  ```
  if X >= max
    score = 100
  if X <= min 
    score = -100
  ```



In the example, we can't know the maximum value of mem and cpu of the cluster, so use the second calculation method, where ```MAXCPUCOUNT``` and ```MAXMEMCOUNT``` are the values we set manually. At the same time, because the two numbers cpuValue and memValue must be positive numbers, the calculation formula can be simplified:  ```score = 100 / max * X```

```go
var availableCpu float64
availableCpu = float64(cpuAllocInt - cpuUsage)
if availableCpu > MAXCPUCOUNT {
  cpuValue = int64(100)
} else {
  cpuValue = int64(100 / MAXCPUCOUNT * availableCpu)
}

var availableMem float64
availableMem = float64((memAllocInt - memUsage) / (1024 * 1024))
if availableMem > MAXMEMCOUNT {
  memValue = int64(100)
} else {
  memValue = int64(100 / MAXMEMCOUNT * availableMem)
}
```

[gif]
