Aug 21, 2020
# Kubernetes Training 4

## AWS Container Day Recap

### Key Take-Aways:
- Plans to increase number of addressable IP addresses, which will allow us to increase the density of pods on each work node.
- AWS sees GitOps as the best practice for deploying containers, especially in production. More on GitOps later.

### New Products Announced
- `cdk8s`: A new CDK for Kubernetes
  - Use common programming languages (like JavaScript) to define K8s objects through code. The code generates standard K8s YAML files.
- AWS Controllers for Kubernetes, or ACK
  - A set of K8s controllers and CRDs for AWS services that allow you to instantiate and use AWS resources directly within cluster. 
  - ACK is a viable alternative to CloudFormation. Allows you to manage AWS resources in one place.
  - See this [YouTube clip](https://www.youtube.com/watch?v=6TWIoGkWEIc&t=1903s).

---

## Autoscaling Applications and Clusters

In Kubernetes, automatic scaling comes in two forms:
- Scaling pods in a deployment
- Scaling the size of the cluster

### Scaling Pods in a Deployment

A **Horizontal Pod Autoscaler (HPA)** is a Kubernetes object used to scale pods in a deployment. The HPA scales the deployment up or down based on pod metrics such as CPU percent utilization.

For example, here is the HPA K8s manifest for the `accounts-api-dep` deployment in the dev account.

```yaml
---
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: accounts-api-hpa
  namespace: accounts-api
spec:
  minReplicas: 1
  maxReplicas: 20
  targetCPUUtilizationPercentage: 50
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: accounts-api-dep
```

> We'll be working in the `accounts-api` namespace, so to make things easier I'll change the default namespace with [`kubens`](https://github.com/ahmetb/kubectx), a utility to switch between K8s namespaces:
> ```bash
> $ kubens accounts-api
> Context "arn:aws:eks:us-east-1:012345678910:cluster/cluster-name" modified.
> Active namespace is "accounts-api".
> ```
> Now, all `kubectl` commands will be applied to the `accounts-api` namespace.

Let's watch the HPA in the `accounts-api` namespace.
```bash
$ kubectl get hpa --watch
NAME               REFERENCE                     TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
accounts-api-hpa   Deployment/accounts-api-dep   0%/50%    1         5         1          21h
```

In a seperate terminal, watch the pods in the `accounts-api` namespace.
```bash
$ kubectl get po --watch
NAME                                READY   STATUS    RESTARTS   AGE
accounts-api-dep-XXXXXXXX-xxxxx   1/1     Running   0          21h
```

There's currently only one pod running. In a third terminal, we'll use [`hey`](https://github.com/rakyll/hey) to generate load and trigger a scaling event.
```bash
$ hey -q=1 -z=1m -c=20 -m GET -H 'Content-Type: application/json' -H 'x-byub-client: demo' -H "Authorization: $ACCESS_TOKEN" https://accounts-api.dev-byub.org/v1/public/users/me
```

After a few seconds, we should start to notice changes to the HPA and pods we've been watching.

We can also `describe` the HPA to view additional information about the autoscaling event.
<details><summary>Show Output</summary>

```bash
$ kubectl describe hpa accounts-api-hpa
Name:                                                  accounts-api-hpa
Namespace:                                             accounts-api
Reference:                                             Deployment/accounts-api-dep
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  1% (1m) / 50%
Min replicas:                                          1
Max replicas:                                          20
Deployment pods:                                       2 current / 2 desired
Conditions:
  Type            Status  Reason               Message
  ----            ------  ------               -------
  AbleToScale     True    ScaleDownStabilized  recent recommendations were higher than current one, applying the highest recent recommendation
  ScalingActive   True    ValidMetricFound     the HPA was able to successfully calculate a replica count from cpu resource utilization (percentage of request)
  ScalingLimited  False   DesiredWithinRange   the desired count is within the acceptable range
Events:
  Type    Reason             Age    From                       Message
  ----    ------             ----   ----                       -------
  Normal  SuccessfulRescale  4m58s  horizontal-pod-autoscaler  New size: 2; reason: cpu resource utilization (percentage of request) above target
```

</details>

---

### Scaling the Cluster

Cluster Autoscaling happens when pods become unscheduleable due to worker nodes running out of resources, or when the number of worker nodes on the cluster is overprovisioned.



### Quick Note on Horizontal vs Vertical Scaling

According to [Wikipedia](https://en.wikipedia.org/wiki/Scalability#Horizontal_(Scale_Out)_and_Vertical_Scaling_(Scale_Up)):
- **Horizontal Scaling or Scaling Out**: means adding more nodes (i.e., pods) to, or removing nodes from, a system (i.e., deployment).
- **Vertical Scaling or Scaling Up**: means adding resources to, or removing resources from, a single node.

#### Quiz

Q1: In the first demonstration, I generated extra load on the `accounts-api` deployment, which caused the number of pods to increase. Was that an example of horizontal or vertical scaling?

<details><summary>Click to see answer</summary>
Horizontal. We increased the number of pods in the deployment.
</details>

Q2: In the second demonstration, we artifically scaled the number of pods causing the Cluster Autoscaler to add an extra worker node. Was that an example of horizontal or vertical scaling?

<details><summary>Click to see answer</summary>
Horizontal. The Cluster Autoscaler added more resources to the cluster so that the deployment could continue scaling horizontally, but the resources allocated for each individual pod remained the same.
</details>
