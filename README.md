
# Prototype of extensible scheduling using resources usage.
We already support [extensible placement scheduling](https://github.com/open-cluster-management-io/enhancements/blob/main/enhancements/sig-architecture/32-extensiblescheduling/32-extensiblescheduling.md), which allows use of [addonplacementscore](https://github.com/open-cluster-management-io/enhancements/blob/main/enhancements/sig-architecture/32-extensiblescheduling/32-extensiblescheduling.md#addonplacementscore-api) to select clusters, but we lack an addonplacementscore that contains cluster resource usage information.

In this repo, I developed an addon through addon-freamwork, this addon is mainly used to collect resource usage information on the cluster, and generate an addonplacementscore under the cluster namespace of the hub.


# Quickstart
## Prepare
You have at least two running kubernetes cluster. One is the hub cluster, the other is managedcluster.

You can create an ocm environment by running `make deploy-ocm`, which will create a hub and two managedclusters for you.
## Deploy

Set environment variables.

```
export KUBECONFIG=</path/to/hub_cluster/kubeconfig> # export KUBECONFIG=~/.kube/config
```

Build the docker image to run the sample AddOn.

```
# get imagebuilder first
go get github.com/openshift/imagebuilder/cmd/imagebuilder@v1.2.1
export PATH=$PATH:$(go env GOPATH)/bin
# build image
make images
export EXAMPLE_IMAGE_NAME=quay.io/open-cluster-management/resource-usage-collect-addon:latest
```

If your are using kind, load image into kind cluster.

```
kind load docker-image  $EXAMPLE_IMAGE_NAME --name cluster_name # kind load docker-image  $EXAMPLE_IMAGE_NAME --name hub
```

And then deploy the example AddOns controller on hub cluster.

```
make deploy-addonplacementscore
```

## What is next

After the deployment is complete, addon will create an addonplacementscore in its own namespace for each managedcluster in the hub.


```
kubectl config use kind-hub
kubectl get addonplacementscore -A
```


After the addonplacementscore is successfully generated, you can use [extensible placement scheduling](https://github.com/open-cluster-management-io/enhancements/blob/main/enhancements/sig-architecture/32-extensiblescheduling/32-extensiblescheduling.md) to select clusters.

### For example

Select a cluster with more memory free.

```
cat <<EOF | kubectl apply -f -
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: placement
  namespace: ns1
spec:
  numberOfClusters: 1
  prioritizerPolicy:
    mode: Exact
    configurations:
      - scoreCoordinate:
          type: AddOn
          addOn:
            resourceName: test-score1
            scoreName: memAvailable
        weight: 1
EOF
```


```
kubectl get placementdecisions -A
```


# Clean up


```
# clean up this addon
make undeploy-addonplacementscore
```