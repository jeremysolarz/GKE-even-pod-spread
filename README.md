# Evenly spreading pods over cluster nodes

## Problem description
Recently I was faced with a problem on GKE.

The issue was regarding uneven scaling of pods across worker nodes. 

## Solutions

### Solution 1: Changing Kube Scheduler settings

Since in a managed offering chaning the config of the master node (as explained in soution #2) are hidden and Kubernetes now have a standard way to do similar without touching the masters. We can do it with using [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/).

To test it out first create a cluster >=v1.19.
```
project=$(gcloud config get-value project)

gcloud beta container --project ${project} clusters create "cluster-1" \
  --zone "us-central1-c" --cluster-version "1.19.12-gke.2101" \
  --enable-ip-alias \
  --release-channel "stable" --num-nodes "1" \
  --node-locations "us-central1-a","us-central1-b","us-central1-c"
```

After the command finishes you should be able to access the cluster via `kubectl`.

Run `kubectl get pods` to check it.

You should get a message like:
```
No resources found in default namespace.
```

Now you can apply the two deployments in this repo.

Run `kubectl apply -f .`.

This should create two deployments. One using `Pod Topology Spread Constraints` called *stress-deployment-balanced*. The other called stress-deployment-unbalanced not using it.

Once they get deployed you can query the node distribution with the below command.

```
kubectl get pods -o wide --no-headers | awk '{print "Deployment: " substr($1,1,28) " ---- Node: " $7}' | sort | uniq -c
```

which should give an output similar to this:

```
   1 Deployment: stress-deployment-balanced-5 ---- Node: gke-cluster-2-default-pool-45c87db8-6wrj
   1 Deployment: stress-deployment-balanced-5 ---- Node: gke-cluster-2-default-pool-87e6c349-k19s
   1 Deployment: stress-deployment-balanced-5 ---- Node: gke-cluster-2-default-pool-9c19e0f3-983l
   1 Deployment: stress-deployment-unbalanced ---- Node: gke-cluster-2-default-pool-45c87db8-6wrj
   1 Deployment: stress-deployment-unbalanced ---- Node: gke-cluster-2-default-pool-87e6c349-k19s
   1 Deployment: stress-deployment-unbalanced ---- Node: gke-cluster-2-default-pool-9c19e0f3-983l
```

As you can see initially the spread is even. But if we upscale the deployments the picture might change.

First the unbalanced one:
`kubectl scale deployment --replicas=11 stress-deployment-unbalanced`

Then the balanced one:
`kubectl scale deployment --replicas=11 stress-deployment-balanced`

If we query again we see a different picture:
```
kubectl get pods -o wide --no-headers | awk '{print "Deployment: " substr($1,1,28) " ---- Node: " $7}' | sort | uniq -c
```

As you can see below now the picture is skewed for the unbalanced one:
```
   3 Deployment: stress-deployment-balanced-5 ---- Node: gke-cluster-2-default-pool-45c87db8-6wrj
   3 Deployment: stress-deployment-balanced-5 ---- Node: gke-cluster-2-default-pool-87e6c349-k19s
   3 Deployment: stress-deployment-balanced-5 ---- Node: gke-cluster-2-default-pool-9c19e0f3-983l
   2 Deployment: stress-deployment-balanced-5 ---- Node: gke-cluster-2-default-pool-a05513d0-3hcn
   4 Deployment: stress-deployment-unbalanced ---- Node: gke-cluster-2-default-pool-45c87db8-6wrj
   3 Deployment: stress-deployment-unbalanced ---- Node: gke-cluster-2-default-pool-87e6c349-k19s
   2 Deployment: stress-deployment-unbalanced ---- Node: gke-cluster-2-default-pool-9c19e0f3-983l
   2 Deployment: stress-deployment-unbalanced ---- Node: gke-cluster-2-default-pool-a05513d0-3hcn
```

#### Explanation

I did that in GKE via the usage of standard Kubernetes functionality.

The container is using Pod Topology Spread Constraints via the `topologySpreadConstraints` config. 
```
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: stress-balanced
```
We spread via `topologyKey: kubernetes.io/hostname` that means we are using one of the standard labels of nodes to form a failure domain. 

Taken from the docs:
```
If two Nodes are labelled with this key and have identical values for that label, 
the scheduler treats both Nodes as being in the same topology. The scheduler tries 
to place a balanced number of Pods into each topology domain.
```

And with that we have the even spread we wanted to have.

### Solution 2: Changing Kube Scheduler settings

In a custom managed offering this could be done via [Scheduling Policies](https://kubernetes.io/docs/reference/scheduling/policies/) where you can set a scheduling policy by running `kube-scheduler --policy-config-file <filename>` or `kube-scheduler --policy-configmap <ConfigMap>` and using the [Policy type](https://kubernetes.io/docs/reference/config-api/kube-scheduler-policy-config.v1/).

In that case you give more weight to the `SelectorSpreadPriority`.

A step by step guide explains it in [1] and an illustrative explanation in [2].

[1] [Create Custom Scheduler on GKE for Pod Spreading](https://blog.searce.com/create-custom-scheduler-on-gke-for-pod-spreading-a23c1641a840)

[2] [How to Keep Your Kubernetes Deployments Balanced Across Multiple zones](https://medium.com/expedia-group-tech/how-to-keep-your-kubernetes-deployments-balanced-across-multiple-zones-dfe719847b41)