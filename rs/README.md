# Kubernetes ReplicaSets

* [Intro](#intro)
* [Creating a replicaset](#creating-a-replicaset)
* [Labels](#labels)
  * [Annotations](#annotations)
* [Describing replicasets](#describing-replicasets)
* [Playing with replicasets](#playing-with-replicasets)
* [Applying changes](#applying-changes)
* [Deleting replicasets](#deleting-replicasets)

## Intro

Ok, so you learned about [pods](../pod/README.md). But you won't manipulate pods directly, instead you will use replicasets.

A [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) (or just `rs`) will make sure that the right number of pods replicas are running. So if for any reason one replica just dies, the replicaset will start a new one.

If we have a look at the [replicaset definition in the API reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#replicaset-v1-apps), we can see that it has almost the same top-level fields as the pod's definition. Almost, because the [Spec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#replicasetspec-v1-apps) field is different: 
- it has a `replicas` field - which obviously is the number of pods replicas we want
- a [selector](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#labelselector-v1-meta) field - more on that later
- and a `template` field, which has `metadata` (again), and another `spec` field, which is in fact a [PodSpec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#podspec-v1-core)

It might look complex, but if we take it step by step, you'll see that there is a logic in it ;-)

## Creating a replicaset

Let's start by creating a new `kubernetes-up-and-running-rs.yml` descriptor file, that we will use to create a first replicaset:
- fill in the `kind` and `apiVersion` - take care that the ReplicaSet kind is part of the `apps` group - you can see it at the top of the [replicaset definition in the API reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#replicaset-v1-apps). So the `apiVersion` should be `apps/v1`.
- give it a `name`, inside the `metadata` section
- we would like to have 2 replicas, so in the `spec` section, you'll need to define the `replicas` field to the right value
- and we would like to use a pod that is similar to the pod we previously deployed (using the [gcr.io/kuar-demo/kuard-amd64:1](https://gcr.io/kuar-demo/kuard-amd64:1) image), so we need to tell Kubernetes to use a `template` to create pods. This template should contain the [PodSpec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#podspec-v1-core). So inside the `template` section, let's copy/paste the `spec` of our (previous) pod.

Once you have something that looks nice, let's use the `kubectl create` command to create the replicaset. Did it worked? It should not ;-) It should have failed with something like:

```
error: error validating "kubernetes-up-and-running-rs.yml": error validating data: ValidationError(ReplicaSet.spec): missing required field "selector" in io.k8s.api.apps.v1.ReplicaSetSpec; if you choose to ignore these errors, turn validation off with --validate=false
```

So, we are missing a field - the `selector` one. Oh yes, I remember, I said I would tell you more about it later. Well, seems like later is now. So the replicaset creates pods based on a template, so that there is always the right number of replicas. But how does the replicaset know how many pods are currently running? And there might be other pods, from other replicasets.

## Labels

ReplicaSets uses [labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) to do the replicaset/pods association. In fact, in Kubernetes, every association between different kinds is done using labels. So what are labels? Just a set of key/values, that you can associate to any Kubernetes object, in their `metadata` field.

For example, to set a few labels on a pod:

```
kind: Pod
apiVersion: v1
metadata:
  name: something
  labels:
    team: best-team-ever
    app: my-app
    env: some-env
spec:
  [...]
```

And then, the Kubernetes API allows you to filter resources based on a **label selector** - which is basically a set of key/values that should matches the one defined in the `metadata` field of the objects.

So here, we need to do 2 things:
- in the `template` field of the replicaset, we need to add a `metadata` section with some labels - just use whatever key/value you want, and you can define multiple labels if you want, but one should be enough. These metadata are applied to all the pods created by the replicaset.
- and we also need to add a `selector` section (in the replicaset's `spec`), that will have a `matchLabels` field containing the same labels that are applied to the pods. This will allow the replicaset to query the Kubernetes API and retrieves only the pods that matches the given labels, and thus know how many pods replicas are running - and if it needs to create more or not.

So, basically, you need to set the same set of labels in both the `selector.matchLabels` and the `metadata.labels` fields. If you don't... well, Kubernetes won't even let you create the replicaset, because it makes no sense to create something that you can't manage.

The solution can be found at [kubernetes-up-and-running-rs.yml](kubernetes-up-and-running-rs.yml). Let's try to create it again:

```
$ kubectl create -f kubernetes-up-and-running-rs.yml
```

and then check if we have a running replicaset (or `rs`):

```
$ kubectl get rs
```

Great, we have 1 replicaset, and we can see that it has 2 replicas. Let's check by listing the pods:

```
$ kubectl get pods
```

And we have 2 pods! You can see that each pod has a random name, in the same format of what we did with the `generateName` field of the pod. And indeed, if we have a look at the descriptor of a pod, with `kubectl get pod kubernetes-up-and-running-xxxxx -o yaml`, we can see the use of the `generateName` field. We can also see our labels.

### Annotations

A little side-note about annotations. If you look at the [API reference for metadata](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#objectmeta-v1-meta), you might notice that there are `labels` and `annotations`. Both are sets of key/values, but with a few differences:

- Labels have a more strict naming convention - it needs to support the DNS naming convention (short names, no special characters), which doesn't apply to annotations
- Labels can be used to lookup objects (using labels selectors), but not annotations

So usually annotations are used to store extra information. Kubernetes uses it in some place to store extra JSON data.

## Describing replicasets

Note that once again, the output of the `kubectl describe` command is pretty nice, if you use it on your replicaset:

```
$ kubectl describe rs kubernetes-up-and-running
```

## Playing with replicasets

Ok, time to play! Let's see if this thing really works: let's delete one pod, and see what happens:

```
$ kubectl delete pod kubernetes-up-and-running-xxxxx
```

(of course, replace the `xxxxx` by the real suffix of a running pod)

Use the `kubectl get` command to check how many pods you now have... nice, isn't it? ;-) And in the output of the `kubectl describe` command, you can see the history of the pods created by the replicaset. You can also run `kubectl get pods -w` while deleting a pod.

You can also try to delete multiple pods at once:

```
$ kubectl delete pods --all
```

But the replicaset will always restart new pods.

Let's play a little with labels. Edit a running pod (with `kubectl edit pod kubernetes-up-and-running-xxxxx`), remove its `labels` field (in the `metadata` section), and save. Check your running pods: you now have 3 pods! This is because the replicaset uses a label selector to find matching pods, and when you removed the labels for the pod, the replicaset couldn't see it anymore - as if the pod was deleted. So it just created a new one.

Let's try to list the pods - as the replicaset see them:

```
$ kubectl get pods -l pod=kubernetes-up-and-running
```

(we used the label defined in the [kubernetes-up-and-running-rs.yml](kubernetes-up-and-running-rs.yml) file. We used the `pod` key, but it could have been anything else)

You can also list all pods with their associated labels:

```
$ kubectl get pods --show-labels
```

Now, let's update our pod that doesn't have any labels anymore, and set the `pod=kubernetes-up-and-running` label back on it (or whatever label you used in the replicaset). We could use `kubectl get` to find the pod name, and then `kubectl edit` on it, but if there are multiple pods, it's a very manual operation. Instead, we can use the `kubectl label` command:

```
$ kubectl label pod -l pod\!=kubernetes-up-and-running pod=kubernetes-up-and-running
```

Here, we select all pod which doesn't have the `pod=kubernetes-up-and-running` label, and set this label on them (the `\` is used to escape the `!` in the shell). Run `kubectl get pods --show-labels` again, to see if it worked... and there are only 2 pods now, instead of 3. Because the replicaset saw 3 pods matching its label selector, and deleted 1, to have only 2 replicas.

So labels are a very powerful feature, because you can easily edit them, and remove pods from managed pools, to debug them for example. Setting good labels is hard and takes time, but it's also very important.

## Applying changes

So far we've used the `kubectl create` command to create new objects (pods, replicasets), and the `kubectl edit` command to... edit them. This workflow is nice when you are learning, or when you are playing in a local/sandbox environment. But how do you apply changes in a production environment? Do you run the `kubectl edit` command manually?

It's time to introduce the `kubectl apply` command. This command can be used to both create and update one or more Kubernetes objects:
- if the object doesn't yet exists (based on its name), it will be created
- otherwise, its configuration will be updated based on the current content of the descriptor file

For example, let's try to update our replicaset to have 3 pods instead of 2. Open your [kubernetes-up-and-running-rs.yml](kubernetes-up-and-running-rs.yml) descriptor file locally, update the value of the `replicas` field to be `3` instead of `2`, and then run `kubectl apply` on it:

```
$ kubectl apply -f kubernetes-up-and-running-rs.yml
```

It should warn you that *kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply*, but it's still going to work - you can check it by describing the replicaset, or listing the pods.

So basically, if you want to use `kubectl apply`, you should use it from the start. Because it works by storing the last "applied" configuration in the `annotations` of the replicaset. You can check it by getting the YAML representation of the replicaset with `kubectl get rs kubernetes-up-and-running -o yaml`: you can see the last applied configuration in JSON format, stored in an annotation.

A classic workflow with Kubernetes, is to store the YAML descriptor files in a git repository, and have some kind of CI job (for example Jenkins) running on each commit, and sending each file through `kubectl apply`, so that any change to the git repository is applied to the Kubernetes cluster.

Just for completeness while we are on the topic of scaling: for a quick manual scaling operation, you can also use the `kubectl scale` command - which is an easy way to edit the value of the `replicas` field of a replicaset:

```
$ kubectl scale rs kubernetes-up-and-running --replicas=2
```

and you should now be back to 2 pods. But note that using this command is not compatible with a `kubectl apply`-based workflow, because any change done using `kubectl scale` will be reverted at the next `kubectl apply`...

One more thing on the scaling topic: auto-scaling. Until now we did manual scaling, but Kubernetes also supports 2 kinds of automatic scaling:
- [horizontal pod autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/): adds more pods when required, based on the current CPU usage
- [vertical pod autoscaling](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler): updates the requested resources of the pods, to give more resources to existing pods, instead of creating new pods

This is outside the scope of this *hands-on*, but it's always good to know that these features exists.

## Deleting replicasets

Now that we've played a little, how can we delete all that? Well, just run the `kubectl delete` command on the replicaset:

```
$ kubectl delete rs kubernetes-up-and-running
```

The good news is that deleting a replicaset will also delete all the pods created by this replicaset. We can check that with the `kubectl get` command:

```
$ kubectl get rs,pods
```

Note how we can list multiple types. And that the result should be empty.

Well, in fact you won't manually create replicasets like that. Instead you will use more high-levels concepts, like [deployments](../deployment/README.md).
