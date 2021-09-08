# Evenly spreading pods over cluster nodes

## Problem description
Recently a customer approached me about a problem with GKE he was facing.

They came across an issue regarding uneven scaling of pods across worker nodes. 

## Solutions

### Solution 1: Changing Kube Scheduler settings

Since in a managed offering chaning the config of the master node (as explained in soution #2) are hidden and Kubernetes now have a standard way to do similar without touching the masters. We can do it with using Pod Topology Spread Constraints

### Solution 1: Changing Kube Scheduler settings

In a custom managed offering this could be done via [Scheduling Policies](https://kubernetes.io/docs/reference/scheduling/policies/) where you can set a scheduling policy by running `kube-scheduler --policy-config-file <filename>` or `kube-scheduler --policy-configmap <ConfigMap>` and using the [Policy type](https://kubernetes.io/docs/reference/config-api/kube-scheduler-policy-config.v1/).

In that case you give more weight to the `SelectorSpreadPriority`.

A step by step guide explains it in [1] and an illustrative explanation in [2].

[1] [Create Custom Scheduler on GKE for Pod Spreading](https://blog.searce.com/create-custom-scheduler-on-gke-for-pod-spreading-a23c1641a840)

[2] [How to Keep Your Kubernetes Deployments Balanced Across Multiple zones](https://medium.com/expedia-group-tech/how-to-keep-your-kubernetes-deployments-balanced-across-multiple-zones-dfe719847b41)