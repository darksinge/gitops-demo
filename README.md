Aug 21, 2020
# Kubernetes Training 4

## AWS Container Day Recap

### Key Take-Aways:
- Plans to increase number of addressable IP addresses, which will allow us to increase the density of pods on each work node.
- AWS sees GitOps as the best practice for deploying containers, especially in production. More on GitOps later.

### New Products Announced
- `cdk8s`: A new CDK for Kubernetes
  - Use common programming languages (like JavaScript) to define K8s objects through code. The code generates standard K8s YAML files.
- AWS Controllers for Kubernetes (ACK)
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
> Now, future `kubectl` commands will be applied to the `accounts-api` namespace.

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

First, change the default namespace to `personalization-api`.
```bash
$ kubens personalization-api
```

To demonstrate cluster autoscaling, we'll artificially scale up the deployment by updating the number of replicas in the `personalization-api` deployment.
```bash
$ kubectl scale --replicas=8 deployment/personalization-api
```

Quickly take note of which nodes are running on the cluster.
```bash
$ kubectl get nodes
```

Start watching the pods in the deployment.
```bash
$ kubectl get po --watch
```
We should see a bunch of the pods in the `PENDING` state. These pods are *unschedulable* because there are not enough resources (currently) on the cluster to create them.

In a seperate terminal window, watch the `cluster-autoscaler` logs:
```bash
$ kubectl -n kube-system logs -f deployment/cluster-autoscaler
```

Within a few seconds, we should see a message that looks something like the following:
```text
I0820 22:25:53.745367       1 scale_up.go:435] Estimated 1 nodes needed in eks-28b9764f-4721-3874-b5ed-848fadb2f5fb
I0820 22:25:53.745475       1 scale_up.go:539] Final scale-up plan: [{eks-28b9764f-4721-3874-b5ed-848fadb2f5fb 2->3 (max: 5)}]
I0820 22:25:53.745514       1 scale_up.go:700] Scale-up: setting group eks-28b9764f-4721-3874-b5ed-848fadb2f5fb size to 3
I0820 22:25:53.745541       1 auto_scaling_groups.go:219] Setting asg eks-28b9764f-4721-3874-b5ed-848fadb2f5fb size to 3
I0820 22:25:53.745696       1 event.go:281] Event(v1.ObjectReference{Kind:"ConfigMap", Namespace:"kube-system", Name:"cluster-autoscaler-status", UID:"d34adc8c-4d0f-4a46-adbc-7bf514e6935a", APIVersion:"v1", ResourceVersion:"8596521", FieldPath:""}): type: 'Normal' reason: 'ScaledUpGroup' Scale-up: setting group eks-28b9764f-4721-3874-b5ed-848fadb2f5fb size to 3
```

> Note: We'll scale the `personalization-api` deployment back down when we get to the section on GitOps.



### Horizontal vs Vertical Scaling

According to [Wikipedia](https://en.wikipedia.org/wiki/Scalability#Horizontal_(Scale_Out)_and_Vertical_Scaling_(Scale_Up)):
- **Horizontal Scaling or Scaling Out**: means adding more nodes (i.e., pods) to, or removing nodes from, a system (i.e., deployment).
- **Vertical Scaling or Scaling Up**: means adding resources to, or removing resources from, a single node.

#### Quiz!

**Q1**: In the first demonstration, I generated extra load on the `accounts-api` deployment, which caused the number of pods to increase. Was that an example of horizontal or vertical scaling?

<details><summary>Click to see answer</summary>
Horizontal. We increased the number of pods in the deployment.
</details>

**Q2**: In the second demonstration, we artifically scaled the number of pods causing the Cluster Autoscaler to add an extra worker node. Was that an example of horizontal or vertical scaling?

<details><summary>Click to see answer</summary>
Horizontal. The Cluster Autoscaler added more resources to the cluster so that the deployment could continue scaling horizontally, but the resources allocated for each individual pod remained the same.
</details>

---

### Managing Resources for Containers

In EKS, the number of pods that can run on a worker node is limited by:
- The number of IP addresses available to the node.
- The amount of resources available on the node (most commonly as CPU and memory).

> Aside: The number of available IP addresses on a worker node is determined by the Elastic Network Interface (ENI) attached to the underlying EC2 instances. For smaller instance types, this can be quite small. For example, a `t3.micro` instance type can have a maximum of 2 ENI's, with 2 IPv4 addresses per ENI, adding up to a grand total of 4 IPv4 addresses per worker node ([documentation here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI)).

There isn't much we can do about the first bullet point. However, we can do something about the second bullet point by specifying resource requests for our containers.

The most common resources to specify in a podspec are CPU and memory (RAM) ([see here](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        resources:
          requests:
            cpu: "250m"
            memory: "64Mi"
          limits:
            cpu: "500m"
            memory: "1500Mi"
        ports:
        - containerPort: 8080
          protocol: TCP
```

> Aside: We can go to [aws.amazon.com/ec2/instance-types](https://aws.amazon.com/ec2/instance-types/) to figure out how much memory and vCPU is available on our worker nodes. The EC2 instance type we're using for the worker nodes are `t3.small`, and we can see that our nodes should each have 2 vCPU and 2GiB of memory.

We'll deploy the `nginx` manfiest file in `./k8s/nginx/deployment.yaml`. It's pretty similar to the above manifest, but has the `requests` section commented out:
```bash
$ kubens default # change back to default namespace, just in case
$ kubectl apply -f k8s/nginx/deployment.yaml
```

List our current nodes to find the newest one, which should have been created when we scaled up the `personalization-api` deployment.
```bash
$ kubectl get nodes
NAME                             STATUS   ROLES    AGE     VERSION
ip-192-168-31-160.ec2.internal   Ready    <none>   2m18s   v1.16.8-eks-e16311
ip-192-168-56-164.ec2.internal   Ready    <none>   25h     v1.16.8-eks-e16311
ip-192-168-9-253.ec2.internal    Ready    <none>   25h     v1.16.8-eks-e16311
```

In this case the new node is `ip-192-168-31-160.ec2.internal`. Let's grab some additional information. Take note of things like total requests and limits.
```bash
$ export NODE='ip-192-168-31-160.ec2.internal'
$ kubectl describe node $NODE
```

The requests for CPU resources should be pretty low, which we can also view in the Kubernetes Dashboard. Let's uncomment the resources section in `deployment.yaml`, redeploy, and see what happens.


### That Was Really Annoying... Enter Fargate

AWS Fargate takes off the load of scaling and resource management. I'll walk through the AWS console to show how to set it up, then we'll scale up a deployment and watch what happens.

1. Show how to set up a Fargate profile
1. Deploy the `nginx` deployment into the `demo` namespace
1. Watch it add a pod on a Fargate instance.
1. Scale the deployment and watch the magic happen.

Pros:
- That was easy.
- Don't have to worry about HPA or Cluster scaling. It's just works.
- Don't have to worry about specifying resource requests or limits.
- Don't have to worry about updating/patching software on EC2 instances.

Cons:
- In my tests, Fargate did not scale nearly as fast as the HPA. This is a pretty big drawback.

Lins:
- [Fargate Pricing](https://aws.amazon.com/fargate/pricing/)

## Helm

Before we can talk about GitOps, we need to quickly touch on Helm.

[Helm](https://helm.sh/docs) is the package manager for Kubernetes. 

Quick demo:
```bash
$ kubectl create ns mysql # create the mysql namespace
$ kubens mysql # change default ns used by kubectl
$ helm repo add stable https://kubernetes-charts.storage.googleapis.com/ # Add the 'stable' repo
$ helm install mysql stable/mysql # use helm to deploy a mysql container
$ kubectl delete ns mysql # clean up
```

### Helm Charts

From the [docs](https://helm.sh/docs/topics/charts/) on charts:
> Helm uses a packaging format called charts. A chart is a collection of files that describe a related set of Kubernetes resources.

So, a chart is a collection of files. The file structure for a chart looks like:
```
mychart/
  Chart.yaml
  values.yaml
  charts/
  templates/
```

I'll walk through the [byubroadcasting/eks-gitops](https://github.com/byubroadcasting/eks-gitops) repository, which has a working example.

## GitOps

Go to [this article](https://www.weave.works/technologies/gitops/) for an explanation of GitOps.

Then go to the [EKS workshop](https://www.eksworkshop.com/intermediate/260_weave_flux/) site for a walkthrough of how to set up GitOps on a cluster.

Links:
- [Flux](https://github.com/fluxcd/flux)
- [Helm operator](https://github.com/fluxcd/helm-operator)

