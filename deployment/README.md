# Kubernetes Deployments

* [Intro](#intro)
* [Creating a deployment](#creating-a-deployment)
* [Describing a deployment](#describing-a-deployment)
* [Deploying new versions](#deploying-new-versions)
* [Rollback](#rollback)
* [Deployment history](#deployment-history)
* [Pausing a deployment](#pausing-a-deployment)
* [Scaling a deployment](#scaling-a-deployment)

## Intro

Ok, so you learned about [pods](../pod/README.md) and [replicasets](../rs/README.md). But you won't manipulate pods or replicasets directly, instead you will use deployments.

A [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) is a wrapper on top of replicasets, that introduce the concept of deployment. While the replicaset was static - no automatic redeployment of the pods if you change something in the replicaset - the deployment brings life to your pods!

A deployment will always try to automatically apply any change to the underlying running pods. So everytime you change something at the deployment level, a new replicaset will be created, which will result in new pods being created (and the old ones deleted).

## Creating a deployment

So let's try to write a deployment descriptor file! As usual, we will start from the [deployment definition in the API reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#deployment-v1-apps), and you will notice how similar it is to the replicaset definition - but with a few more optional fields. That means that we can take a simple replicaset definition, change its type to `Deployment`, and we're done!

As usual, try to create the deployment with the `kubectl create` (or `kubectl apply`) command. The solution can be found at [kubernetes-up-and-running-deployment.yml](kubernetes-up-and-running-deployment.yml). After the deployment creation, we now expect the output of the `kubectl get` command to tell us that we have one deployment:

```
$ kubectl get deployment
```

If you also check your replicasets and pods (still with the `kubectl get` command), you will see that the deployment has already created an replicaset (with a randomly generated name), and that in turn the replicaset has created some pods.

## Describing a deployment

We can have a closer look at the deployment with the super cool `kubectl describe` command:

```
$ kubectl describe deployment kubernetes-up-and-running
```

## Deploying new versions

So let's try to deploy a new version of our docker image: edit the deployment (using `kubectl edit`, or `kubectl apply` on your local changes) and replace [gcr.io/kuar-demo/kuard-amd64:1](https://gcr.io/kuar-demo/kuard-amd64:1) with [gcr.io/kuar-demo/kuard-amd64:2](https://gcr.io/kuar-demo/kuard-amd64:2). Wait a few seconds. Then have a look at a running pod:

```
$ kubectl describe pod kubernetes-up-and-running-xxxxxxxxx-yyyyy
```

And you should see that it has the right version (`2`) of the docker image. So when we updated the deployment, it deleted the pods running the version 1 of the image, and created new ones using the version 2 of the image.

If you `kubectl describe` again the deployment, you can some events in the bottom of the output, with some interesting information about scaling up and down replicasets. It's because the default strategy to redeploy is to use a **rolling restart**:
- the deployment creates a new replicaset, with `0` replicas
- it then scales up this new replicaset to `1` replica
- and scales down the old replicaset from `2` to `1` replica
- and so on, until the old replicaset is scaled down to `0` replicas

We can confirm that by looking at the replicasets:

```
$ kubectl get rs
```

You can see 2 replicasets, 1 with `0` replicas, and another one with `2` replicas. So the previous replicaset doesn't change - except for the number of replicas.

## Rollback

This allows for easy rollbacks: it's just a matter of scaling up the old replicaset, and scaling down the new replicaset. But it's not something you should do manually - instead, use the `kubectl rollout undo` command:

```
$ kubectl rollout undo deployment kubernetes-up-and-running
```

and then check again the replicasets: the first one now has 2 replicas, and the second one doesn't have any replica.

## Deployment history

You can use the `kubectl rollout history` command to view the history of changes:

```
$ kubectl rollout history deployment kubernetes-up-and-running
```

but the really interesting feature is the diff between 2 revisions, using the `--revision` flag:

```
$ kubectl rollout history deployment kubernetes-up-and-running --revision=2
```

this shows the diff between revision 1 and 2, when we upgraded the docker image version from 1 to 2.

While we are talking about history, let's talk about history limit: how many "old" replicasets are kept by Kubernetes? How far can you rollback? This is defined by the `revisionHistoryLimit` field in your [Deployment's Spec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#deploymentspec-v1-apps), and it has a default value of `10`.

So if you don't want to see too many old replicasets, and you only care about keeping 1 revision history, you can set it to `1`. And if you want to disable rollback completely, you can set it to `0`.

For example, let's try to do a new deployment, with:
- no revision history - so the `revisionHistoryLimit` value set to `0`
- the [gcr.io/kuar-demo/kuard-amd64:3](https://gcr.io/kuar-demo/kuard-amd64:3) docker image

Either `kubectl edit` your deployment, or update it locally and `kubectl apply` it. Wait a little, and check your replicasets or your deployment history: you should now have only 1 replicaset - the current one - and a single revision history.

## Pausing a deployment

You can also use the `kubectl rollout` command to pause/resume a deployment: a paused deployment won't do anything when its configuration changes. You will need to resume it if you want your pods to be redeployed.

One more thing: if your deployment is a little slow (lots of replicas, or slow startup, ...), you can follow the deployment status using the `kubectl rollout status` command.

## Scaling a deployment

A deployment can be scaled up/down the same way a replicaset is scaled: by editing the deployment's `replicas` field, or by using the `kubectl scale` command:

```
$ kubectl scale deployment kubernetes-up-and-running --replicas=3
```

Ok, so now we have 3 running pods. Maybe it's time to actually use them, and access our web application through a [service](../svc-ep/README.md)!
