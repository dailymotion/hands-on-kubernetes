# Kubernetes Pods

* [Intro](#intro)
* [Creating a pod](#creating-a-pod)
* [Listing pods](#listing-pods)
* [Describing pods](#describing-pods)
* [Accessing logs](#accessing-logs)
* [Connecting to a pod](#connecting-to-a-pod)
* [Editing a pod](#editing-a-pod)
* [Deleting pods](#deleting-pods)
* [Creating multiple pods](#creating-multiple-pods)
* [Watching pods](#watching-pods)
* [Pods resources](#pods-resources)
* [Pods events](#pods-events)

## Intro

A [pod](https://kubernetes.io/docs/concepts/workloads/pods/) is a collection of (Docker) containers that share some resources - mainly the same network and file system. It comes from the observation that an application can be composed of different parts (proxy, frontend, backend, ...), each part being a container. And so it is easier to manipulate all those containers as one single entity: a Pod. So in Kubernetes, the basic entity that is deployed is a Pod.

## Creating a pod

Let's create some pods! For that, we will use the `kubectl create` command. Let's start with its documentation:

```
$ kubectl create -h
```

So it tells us that we need a YAML or JSON file that represents the resource that we want to create. Usually people write YAML files for Kubernetes, but if you prefer JSON, that's ok - however note that everything in this *hands-on* will use YAML files.

We can find the definition of the resource types in the [Kubernetes API Reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/). The [Pod](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#pod-v1-core) reference is accessible directly from the left menu. The top-level fields are:

- `kind`: each resource has a `kind` (its type): `Namespace`, `Pod`, `Service`, ... Remember that the list of all supported kinds is available using `kubectl api-resources` (see the `Kind` column...)
- `apiVersion`: we also need to specify the API version - for now, this is `v1`, except for resources that came later (beta features for example)
- `metadata`: the `name` field of the `metadata` is required - it is the name of the resource
- `spec`: this is where we can define each container that will be in our pod: its name, its docker image, ...

An alternative to the online documentation is to use the embedded documentation, using the `kubectl explain` command:

```
$ kubectl explain pod
```

Or to get the documentation of a sub-field: `kubectl explain pod.spec` or even `kubectl explain pod.spec.containers.image`.

The first exercice is to write a `kubernetes-up-and-running-pod.yml` file that will be used to create a pod with a single container based on the [gcr.io/kuar-demo/kuard-amd64:1](https://gcr.io/kuar-demo/kuard-amd64:1) docker image (it is the [companion application](https://github.com/kubernetes-up-and-running/kuard) that comes with the [Kubernetes Up and Running book](https://www.amazon.fr/dp/B075G373MJ/ref=dp-kindle-redirect?_encoding=UTF8&btkr=1) - a must-read, btw). You can start with the following snippet, and write the `spec` field value based on [its definition](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#podspec-v1-core):

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: kubernetes-up-and-running
spec:
  [...]
```

Once you have something you feel comfortable with, it's time to create your pod !

```
$ kubectl create -f kubernetes-up-and-running-pod.yml
```

If it fails, fix your descriptor and try again ;-) The solution can be found at [kubernetes-up-and-running-pod.yml](kubernetes-up-and-running-pod.yml)

## Listing pods

Good, so now you should have a running pod! You can see it by running:

```
$ kubectl get pods
```

Ok, time for a big side note about the `get` command, and the others CRUD commands. Kubernetes has quite a few types of resources (you can list them with the `kubectl api-resources` command), and the basic operations you will want to do on these resources are CRUD: Create/Read/Update/Delete. And there are commands mapped on these actions: `kubectl get`, `kubectl describe`, `kubectl edit`, `kubectl delete`. All those commands work in the same way: `kubectl COMMAND RESOURCE [NAME]`. So once you know how `kubectl get` works, it's mainly the same thing for its friends ;-)

So while `kubectl get pods` will get you the list of pods, you can also get a single pod with:

```
$ kubectl get pod kubernetes-up-and-running
```

So, the output of the previous command is... not so much interesting, isn't it? But the `kubectl get` command has a lot of nice options, that you can view with:

```
$ kubectl get -h
```

For example, you can get a YAML representation of the pod with:

```
$ kubectl get pod kubernetes-up-and-running -o yaml
```

(and the JSON-lovers can use `-o json`)

We can see that the YAML definition is now quite more complex than what we gave to Kubernetes! This is because we just wrote the minimum required fields to have a valid descriptor, but then Kubernetes added some default values - and there are quite a few of them. You can go through the [Pod definition in the Kubernetes API reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#pod-v1-core) to read the documentation for some of those fields - for example the `imagePullPolicy` or `restartPolicy` ones.

## Describing pods

Another nice alternative of the `kubectl get` command for a single resource is the `kubectl describe` command:

```
$ kubectl describe pod kubernetes-up-and-running
```

Nice, isn't it? I knew you would love it ;-) The `describe` output is made for humans, and aggregates data from multiple sources to present you a nice view.

## Accessing logs

Now let's have a look at the logs, with the `kubectl logs` command:

```
$ kubectl logs kubernetes-up-and-running
```

We can see that the `kubernetes-up-and-running` application in our container listen on port `8080`.

Side node about logging: we can see the application's logs that way, because it writes logs to `stdout`, which is the recommanded way to write logs in containers - see [12-factor apps](https://12factor.net/logs) for more on that topic. So basically `kubectl logs` will connect you to the `stdout` of your container.

## Connecting to a pod

Now, let's try to make an HTTP request to the application, to make sure that it's really deployed and available. We will use the `kubectl port-forward` command to forward a local port to a remote port on our pod:

```
$ kubectl port-forward kubernetes-up-and-running 8080
```

This command will forward the local port `8080` to the remote port `8080` on the `kubernetes-up-and-running` pod. So if we just open <http://localhost:8080/> we can send an HTTP request to our application - and see the response.

Allright, that was easy: just one simple container in a pod. What about adding more containers? We will try to run 2 instances of the [gcr.io/kuar-demo/kuard-amd64:1](https://gcr.io/kuar-demo/kuard-amd64:1) docker image in the same pod:

write a `duplicate-containers.yml` file for a pod named `duplicate-containers`, and 2 containers named `one` and `two` based on the [gcr.io/kuar-demo/kuard-amd64:1](https://gcr.io/kuar-demo/kuard-amd64:1) docker image. The solution can be found at [duplicate-containers.yml](duplicate-containers.yml) (but try to write it by your own first - that's the best way to learn)

Once it's done, create the pod (`kubectl create -f ...`), and have a look at its status with the `kubectl describe` command. You should notice that while a container is running fine, the other kept restarting (check the `Restart Count`). So, let's have a look at the logs, with the `kubectl logs` command... but it's not working now: Kubernetes is asking us for a container name! Back to the command's help with `kubectl logs -h`, you will learn that we need to specify a container name, unless there is only 1 container in the pod.

Note that if you did setup the CLI completions, you can auto-complete the pod name and the containers names. Nice, isn't it? And if you didn't take the time to setup the completions, maybe you might want to reconsider now? ;-)

So the logs of the container `one` - retrieved with `kubectl logs duplicate-containers -c one` - are what we would expect:

```
Serving on HTTP on :8080
```

While the logs of the container `two` - retrieved with `kubectl logs duplicate-containers -c two` - are not what we expected:

```
Serving on HTTP on :8080
listen tcp :8080: bind: address already in use
```

Note that if the only output you get from the `kubectl logs` command is:

```
Error from server: Internal error occurred: Pod "duplicate-containers" in namespace "hands-on": container "two" is in waiting state.
```

You just happened to have requested the container's logs while it was down. But don't worry, because the `kubectl logs` has a nice option to help you with that: the `kubectl logs -p` option can get you the logs of a previous container instance. So you just need to retry with `kubectl logs duplicate-containers -c two -p`.

It is pretty clear from the logs output that the second container can't bind to the port `8080` because it is already in use - by the first container! If you are used to Docker this would seems strange to you, because you would expect each container to have its own network namespace, but remember that we are in a pod! And in a pod, all the containers share the same network namespace, as if all the applications were deployed on the same host.

So, we need a way to tell 1 of the 2 containers to run on a different port. Maybe the application supports some flags, so we can configure it easily? If only we could SSH into the container and run the `--help` flag on the binary...

Fortunately, Kubernetes has the `kubectl exec` command, that allows you to run a command inside a running container! You can see how it works by running `kubectl exec -h`. For example, to start a Shell script on the container `one` of the `duplicate-containers` pod:

```
$ kubectl exec -it duplicate-containers -c one sh
```

So, we are now in a Shell inside the container. If you run `ls` to inspect the filesystem, you can see the `kuard` binary. Guess that's what we're looking for ;-) So let's see the list of supported flags by running `./kuard -h`. Hummm, `--address`, with a default value of `:8080`. That's what we need to change!

## Editing a pod

So we will configure the arguments passed to the binary when starting the second container, to use a different port. If we look at the [Container definition in the Kubernetes API reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#container-v1-core), we can see that there is an `args` field, that can be set to an array of arguments to pass to the entrypoint. Let's try to edit the pod spec with the `kubectl edit` command:

```
$ kubectl edit pod duplicate-containers
```

This should open an editor with the YAML representation of the pod. Add an `args` field to the [Container definition](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#container-v1-core) for container `two`. The solution can be found at [duplicate-containers-with-custom-port.yml](duplicate-containers-with-custom-port.yml) (don't worry if you have more fields when you run the `kubectl edit` command: those other fields are auto-generated from default values) Save, and... you should get the following error:

```
error: pods "duplicate-containers" is invalid
```

and in the editor, in can see a more explicit error message:

```
pods "duplicate-containers" was not valid:
* spec: Forbidden: pod updates may not change fields other than `spec.containers[*].image`, `spec.initContainers[*].image`, `spec.activeDeadlineSeconds` or `spec.tolerations` (only additions to existing tolerations)
```

Ok, so it seems like Kubernetes really doesn't want us to update the definition of a running pod (that's why we are going to use replicasets and deployments - but more on that later). We'll need to delete it, and re-create it.

But before, time for another side note about the editor used to edit resources. Editing resources on Kubernetes is something you will do a lot, so you might want to use an editor you are familiar with. For example, I prefer to use [Visual Studio Code](https://code.visualstudio.com/) to edit resources, so I defined it as my Kubernetes editor with the `KUBE_EDITOR` environment variable:

```
export KUBE_EDITOR='code -w'
```

(the `-w`, or `--wait` flag, is used to "wait for the files to be closed before returning", which is exactly what we want, because once the editor returns, `kubectl` will send the content of the file to the API server)

So, back to our pods: let's just remove this pod with the `kubectl delete` command:

```
$ kubectl delete pod duplicate-containers
```

and re-create it from the [duplicate-containers-with-custom-port.yml](duplicate-containers-with-custom-port.yml) file, with the `kubectl create` command:

```
$ kubectl create -f duplicate-containers-with-custom-port.yml
```

and make sure that this time it is working. The `kubectl describe` command should tell you that the second container has the right arguments, and if you have a look at the logs you can confirm that the application is listening on the right port.

## Deleting pods

Now let's delete our `duplicate-containers` pod. We can use the `kubectl delete` command again:

```
$ kubectl delete pod duplicate-containers
```

and then check with `kubectl get pods` that it is gone. You should have only 1 pod: the `kubernetes-up-and-running` pod.

## Creating multiple pods

Now, what if we wanted to deploy more than 1 instance of our `kubernetes-up-and-running` pod? Let's see if we can just create a second pod, based on the same `kubernetes-up-and-running-pod.yml` descriptor file. Because it's a different pod, we shouldn't have any port-conflict issue.

```
$ kubectl create -f kubernetes-up-and-running-pod.yml
```

And... no! we can't just create multiple instances of the same pod, because each name needs to be unique. So what is the solution? Hack the descriptor file to change the name each time we want to deploy a new instance of the same pod? Or find an alternative in the [Pod's metadata documentation](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#objectmeta-v1-meta)? So let's create a new descriptor file `kubernetes-up-and-running-generic-pod.yml` but with a different `metadata` section, so that we can deploy the same pod multiple times. The solution can be found at [kubernetes-up-and-running-generic-pod.yml](kubernetes-up-and-running-generic-pod.yml). Then, create a first pod based on this new file:

```
$ kubectl create -f kubernetes-up-and-running-generic-pod.yml
```

Check with the `kubectl get pods` command... You now have a new pod with a random name, thanks to the `generateName` field. So we should now be able to create more than 1 instance of this pod! Go, test it by running multiple times the latest `kubectl create` command, and check the result with the `kubectl get pods` command.

## Watching pods

You can even see live the new pods being deployed, if you "watch" for changes, using the `-w` (or `--watch`) flag on `kubectl get`: in a terminal, run the following command:

```
$ kubectl get pods -w
```

It will list the current pods, but won't return right away. Leave it running, and in another terminal, create one more pod, and then come back to the previous terminal. You should see new output, with a new pod name, and different statuses each time: that's because Kubernetes will tell you each time the pod has changed.

But in fact you won't manually create pods like that. Instead you will use more high-levels concepts, like the [replicaset](../rs/README.md), that we will see in the next section.

## Pods resources

But before, we'll look at one more features of the pods: the resources. You might know that one of the benefits we get from containers is better resources management: you can limit the resources used by a specific container, so that it doesn't take all the resources of the underlying host.

Kubernetes allows you to define both:
- **requested** resources: the minimum (CPU / memory / ...) resources that your container will need to run. Kubernetes will use this to deploy your pod on a host with sufficient available resources.
- **limit** resources: the maximum (CPU / memory / ...) resources that your container will be able to use.

You can read more about resources management in the [Kubernetes documentation](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/).

So, let's play with resources, and we'll start with limits: we'll write a new YAML file based on the `kubernetes-up-and-running-generic-pod.yml` file, but we a memory limit of 256 MB (`256M`) for the container. Note that the resources are defined at the container-level, not at the pod-level. You can find the solution in the [kubernetes-up-and-running-pod-with-limits.yml](kubernetes-up-and-running-pod-with-limits.yml) file.

Create the pod, and check the output of the `kubectl describe` for example, to make sure that the new settings have been applied. You should see

```
Containers:
  kuar:
    Limits:
      memory:  256M
    Requests:
      memory:  256M
```

The first interesting thing is that while we didn't specify any "requested" resources, it seems like some have been applied for us. That's because if you ask for some limit, but no requests, Kubernetes will set the requests to the value of the limit.

Now, let's see what happens if we eat more than 256 MB of memory. Use the `kubectl port-forward` command to connect to the pod's HTTP port, navigate to the **Memory** page at <http://localhost:8080/-/mem>, and click on the "Allocate 500 MiB" link. Hum... something happens, like a connection issue warning, but then... nothing. So what did really happen ? The output says we're still using only a few MB. Is there a bug in the application? Or in Kubernetes?

Let's try to list our pods again, with `kubectl get pods`. Notice something? The **RESTARTS** column has a value of `1` for our pod. So it did restart for some reason...

Let's run the `kubectl get pods -w` command to see what's happening, and click again on the "Allocate 500 MiB" link in the application's UI. This time we can see something really interesting in the output of `kubectl`:

```
NAME                                          READY     STATUS            RESTARTS   AGE
kubernetes-up-and-running-with-limits-clb5k   0/1       OOMKilled         1          32s
kubernetes-up-and-running-with-limits-clb5k   0/1       CrashLoopBackOff  1          43s
kubernetes-up-and-running-with-limits-clb5k   1/1       Running           2          44s
```

So our container got killed by the OOM (Out-Of-Memory) Killer, which is responsible for killing processes that eat too much memory. Kubernetes than restarted it. This is what happens if you're trying to use more memory than the configured limit. For CPU, well, you just won't be able to use more CPU than configured, but it won't restart your container.

Now, let's play with requests instead of limits: write another YAML file, but this time with a request of 32 CPUs for example (something big enough). You can find the solution in the [kubernetes-up-and-running-pod-with-requests.yml](kubernetes-up-and-running-pod-with-requests.yml) file.

Create the pod, and check the output of the `kubectl describe` for example, to make sure that the new settings have been applied. The first thing your might notice, is that the output is much smaller than for other pods. And it's status is set to `Pending`. If you look at the bottom of the output, you can see an **Events** section:

```
Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  13s (x6 over 28s)  default-scheduler  0/1 nodes are available: 1 Insufficient cpu.
```

So clearly our pod couldn't be scheduled to a node, because Kubernetes couldn't find a node with sufficient resources (CPU). It will retry until either the finds a node with enough resources, or the pod is killed. There are usually 2 ways to find a node with enough resources:
- delete running pods, until there is enough free resources for the new pod to run
- or add a new node. This is usually a manual operation, but on Cloud-operated Kubernetes clusters, for example GKE, it can be done automatically: you setup a node-pool with the auto-scaling feature enabled, and when Kubernetes will fail to schedule a pod on an existing node, it will try to create a new node.

This is why it is very important to always define requests and limits for both CPU and memory, with something like:

```
spec:
  containers:
  - name: ...
    resources:
      requests:
        cpu: 500m
        memory: 256M
      limits:
        cpu: 2
        memory: 1024M
```

Not setting the right requests/limits resoureces on your pods will not only affect you, but also all other users of the Kubernetes cluster.

Note that in this *hands-on* we won't define the pods resources, to keep descriptors as minimal as possible, but in real life you should always do it. And some IDE integrations will warn you if you don't do it.

## Pods events

One more thing before closing this chapter on pods: we just saw that there are pod-related events. In fact, in Kubernetes, there are events for everything. You can see them as any other resource, using `kubectl get`:

```
$ kubectl get events
```

That's a good way to see what is happening - and something you will use often to understand why something didn't happened the way you expected it to happen! Just remember that this command list all the events, this is why the `kubectl describe` is so nice: it automatically filters only the events related to the pod you are interested in.

So, time for cleanup before we move on. Let's remove all the pods we created in the current namespace:

```
$ kubectl delete pods --all
```

And get on with the next step: have some fun with [replicasets](../rs/README.md)!
