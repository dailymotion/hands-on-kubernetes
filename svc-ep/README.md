# Kubernetes Services and Endpoints

* [Intro](#intro)
* [Creating a service](#creating-a-service)
* [Listing services](#listing-services)
* [Describing services](#describing-services)
* [Understanding services](#understanding-services)
  * [Endpoints](#endpoints)
* [Accessing services](#accessing-services)
  * [From inside the cluster](#from-inside-the-cluster)
  * [From outside of the cluster](#from-outside-of-the-cluster)
* [Probes](#probes)
* [Labels](#labels)
* [Service Internals](#service-internals)

## Intro

So far we had a good overview of the workflow to deploy an application, but we still don't know how to access our application. To do that, we first need to introduce the concept of services. And endpoints.

A [Service](https://kubernetes.io/docs/concepts/services-networking/service/) (or just SVC) is an entrypoint that can be used to access pods. While the pods are "dynamic" (they can die and be restarted), the service will stay static, so that we can easily access it. And the service is in charge of forwarding the requests to the active pods.

## Creating a service

Let's try something different to create our service. Sure, we could write the descriptor file manually, based on the [service definition in the API reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#service-v1-core), and then use either `kubectl create` or `kubectl apply` to create it. But aren't you tired of always doing the same thing?

I am, so let's use a different approach this time: we'll use an helper command built-in `kubectl` that will do the work for us: `kubectl expose`. First, let's use the internal help to know a little more about it:

```
$ kubectl expose -h
```

As you can see, you can do lots of things with this command. Let's start simple, and just expose our existing `kubernetes-up-and-running` deployment as a service:

```
$ kubectl expose deployment kubernetes-up-and-running
```

And... that fails. It seems it can't find which port to expose. So we can either tell the `kubectl expose` command which port to expose explicitly using the `--port` flag, or we can follow best practices, and declare the ports in our container definition, and let `kubectl` do its introspection automatically.

So, let's edit our existing deployment, and add the `ports` inside the [container](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#container-v1-core). Remember that the application is running on port `8080` by default. The solution can be found at [kubernetes-up-and-running-deployment-with-ports.yml](kubernetes-up-and-running-deployment-with-ports.yml). It's up to you if you want to use `kubectl edit` or `kubectl apply` - you're a grown up now, it's time to pick your own workflow ;-)

Once it's done, you can make sure the change was applied, for example by describing a running pod. Then, try to run the `kubectl expose` command again:

```
$ kubectl expose deployment kubernetes-up-and-running
```

This time, it should work.

## Listing services

Let's make sure our service is there, by listing them. I'm lazy, so I'm using the shortname `svc`, instead of `service`. Remember that you can get the list of all kinds and their associated shortnames by running `kubectl api-resources`.

```
$ kubectl get svc -o wide
```

I'm also using the *wide* output to display the label selector directly. Because services uses labels and label selector to find the targeted pods - the same way replicasets works.

If we want to see the descriptor file we would have to write manually if we didn't used the `kubectl expose` command:

```
$ kubectl get svc ubernetes-up-and-running -o yaml
```

## Describing services

Let's describe our service to understand how it works, and what we can do with it:

```
$ kubectl describe svc kubernetes-up-and-running
```

Well, at least it's a short output, it must be easy to understand.

## Understanding services

We can see that our service has an IP, a port, and 2 endpoints. Each endpoint has its own IP. Let's make a guess: is that the IPs of the pods? Let's display our pods with their IPs:

```
$ kubectl get pods -l pod=kubernetes-up-and-running -o wide
```

I'm using the label selector to retrieve only the "matching" pods (even if I know that there are no other pods), and the *wide* output to list the IPs for each pod. And indeed, each endpoint IP is the IP of a pod.

### Endpoints

In fact, the endpoints are a resource on their own. So we can list them with:

```
$ kubectl get ep
```

(`ep` is the shortname for `endpoints`) - or you can get more details with either:

```
$ kubectl get ep kubernetes-up-and-running -o yaml
$ kubectl describe ep kubernetes-up-and-running
```

Let's try to scale our deployment up to 3 pods instead of 2, using

```
$ kubectl scale deployment kubernetes-up-and-running --replicas=3
```

and check the endpoints again: you can now see 3 IPs, for our 3 pods. So each pod creation/deletion is reflected in the endpoints, which is used by the service to target the right pods.

## Accessing services

So, accessing our web application should be as easy as opening <http://service_ip:port>, isn't it? Well, you can try it. Don't wait too long for an answer, because it shouldn't answer. Because this IP is a private IP, that is only accessible from inside the cluster. And most likely you are either running a Kubernetes cluster in a VM, or operated by a cloud provider. So you can't reach this private IP.

### From inside the cluster

This is because by default, services are made to expose pods to other pods inside the cluster. Let's try it by opening a shell inside one of the pods:

```
$ kubectl exec -it kubernetes-up-and-running-xxxxxxxxx-yyyyy /bin/sh
```

and once you're inside the shell, try again to make an HTTP request to the web application (don't forget to use the right IP address of the service):

```
$ wget http://aa.bb.cc.dd:8080/ -O /dev/stdout
```

It should return you the HTML content of the homepage. And if you don't want to remember IP addresses, you can use the internal DNS, which is setup to automatically expose the services:

```
$ wget http://kubernetes-up-and-running:8080/ -O /dev/stdout
```

So the DNS server will return the service IP for the following names:
- `kubernetes-up-and-running` (the service name)
- `kubernetes-up-and-running.hands-on` (the service name with the namespace as suffix - for example if you want to access a service from another namespace)
- `kubernetes-up-and-running.hands-on.svc.cluster.local` (the same, but with the full cluster optional suffix)

### From outside of the cluster

Great, so we know how to access the service from inside the cluster, but we still can't access it from outside. Time to talk about **service type**: a service can be of different type, each one with different features:
- **ClusterIP**: this is the default - if you inspect our current service, you will see that it is using the `ClusterIP` type. A ClusterIP service is only exposed inside the cluster, using a private IP.
- **NodePort**: has all the features of the ClusterIP type - which is just an internal IP - and in addition, is also exposed through a port opened on each node of the cluster. This new feature allows you to access your service from outside of the cluster: if you can access your cluster's nodes, then you can access your service.
- **LoadBalancer**: has all the features of the NodePort type - private IP + node port - and in addition, can also integrate wîth a cloud provider to automatically create a load balancer that targets the node ports.

Let's start by trying to update our existing service to use the `NodePort` type instead of the `ClusterIP` one. We don't have the descriptor file locally, so obviously we can't use `kubectl apply` here. Want to use `kubectl edit`? No, let's do something different: use `kubectl patch`. First, check the inline doc:

```
$ kubectl patch -h
```

So this command allows us to change a field just by providing a JSON snippet. Let's first display the current service descriptor using `kubectl get svc kubernetes-up-and-running -o yaml`, and notice that the path of the field we want to change is `spec.type`. That should be easy:

```
$ kubectl patch svc kubernetes-up-and-running -p '{"spec":{"type":"NodePort"}}'
```

Then describe your service again, and notice that it now has a `NodePort` filled with some random port.

So now we should be able to access our service, right? Well, that depends...
- if you are on a Mac, using Kubernetes setup through [Docker for Mac](https://docs.docker.com/docker-for-mac/), yes, you can hit <http://localhost:NODE_PORT/> and see the web application! This is because the ports opened on the Docker VM are also available through your local interface.
  - you can use the following command to retrieve only the node port from the descriptor:
    ```
    $ kubectl get svc kubernetes-up-and-running -o=jsonpath='{.spec.ports[0].nodePort}'
    ```
  - so you can run the following command to open the web UI directly in your browser, from your terminal:
    ```
    $ open http://localhost:$(kubectl get svc kubernetes-up-and-running -o=jsonpath='{.spec.ports[0].nodePort}')
    ```
- if you are using [Minikube](https://github.com/kubernetes/minikube), then you can use `minikube ip` to get the IP, and retrieve the port from the service.
- if you are on a cloud-provider hosted Kubernetes cluster... well, your nodes will probably be using private IPs, so you will first need to somehow expose them to the outside world.

This is where the `LoadBalancer` type comes into action: it's made for Kubernetes cluster hosted on a cloud-provider. If you change your service's type again, this time for a LoadBalancer one, you will see that your service will have a new "LoadBalancer Ingress" field, set to some IP/hostname depending on where you are running your Kubernetes cluster. This is the endpoint of an external load balancer (run outside Kubernetes), that is configured to route traffic to your node's IP, on the service port.

So which service type should you use?
- if you don't need your application to be accessible from internet, just use the default `ClusterIP`
- if you need your application to be accessible from internet, use the `LoadBalancer`

## Probes

But a service does more than just forwarding TCP packets to the right pod: it can also check the health or your application, and stop sending traffic to an unhealthy application. The same way you can configure an healthcheck on a load balancer.

To do that, services uses probes, defined at the [container](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#container-v1-core) level. In fact, there are 2 kinds of probes:
- **livenessProbe**, that will be used to check if the container is *live*, meaning if it was able to start correctly, and is running correctly. If not, Kubernetes will just restart it.
- **readinessProbe**, that will be used to check if the container is *ready* to serve traffic. If not, Kubernetes will not send traffic to it.

As you can see in the [probe definition in the API reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#probe-v1-core), you have different ways to check a probe:
- using a **TCP** check on a port
- using an **HTTP (GET)** check on a port
- using a custom **exec** check, using a script/binary on the container

If you have a look at our web application - now that we now how to access it, it's time to use it - we can see that there are both "Liveness Probe" and "Readiness Probe" links in the left menu:
- the liveness HTTP endpoint is exposed at `/healthy`
- the readiness HTTP endpoint is exposed at `/ready`

So let's update our deployment once more, to add both probes to our container. The solution can be found at [kubernetes-up-and-running-deployment-with-probes.yml](kubernetes-up-and-running-deployment-with-probes.yml) (notice that we used the port's name instead of its value, but both works)

Once it's done, describe an instance of a newly deployed pod, to make sure the liveness and readiness probes are well defined. You can also open the web application's UI, and check the liveness/readiness probe pages, which shows the incoming requests from Kubernetes itself. You can see that it's making HTTP requests every 10 seconds, which is the default value for the `period` field. You can try to change it if you want.

But let's play with our readiness probe and our service. You might have noticed that on the "Readiness Probe" page of the webapp, you can manually set the probe to fail. Do it (click on the `Fail` link), wait a little (30 seconds) and then describe your service again. It should have 1 less endpoint than what it used to have. You can also describe the endpoints to have more details:

```
$ kubectl describe ep kubernetes-up-and-running
```

And we can see that 1 IP is marked as not ready. Let's re-enable the pod, by clicking on the `Succeed` link in the UI, and describe the endpoints again: this time all IPs are marked as ready. Pretty nice, isn't it?

If you try the same thing on the liveness probe, and watch for pods changes - using `kubectl get pods -w`, then after a few seconds you will see a pod changing it's "ready" status: from `1/1` containers ready, to `0/1` and then again to `1/1`. That means that a container was restarted. We can see it if we describe this particular pod: in the events section, it clearly says that the liveness probe failed, and that the container was restarted.

This is why it's very important to always define the liveness/readiness probes: if they test the right things, they allow for automatic restart and avoid sending traffic to a non-operational application.

## Labels

We saw that a service use a label selector to find pods to which it should forward traffic. This means that if we update the labels of a pod, the service won't "see" it anymore, and won't send traffic to it anymore. So let's try that, by using the `kubectl label` command:

```
$ kubectl label pod kubernetes-up-and-running-xxxxxxxxxx-yyyyy pod- debug=true
```

Of course don't forget to use the real name of a running pod. So what we just did is:
- remove the `pod` label from the pod - using the `pod-` syntax, see `kubectl label -h` for the supported features
- add the `debug=true` label

Now, this pod shouldn't get anymore traffic. We can check that by describing our service for example, and making sure that the pod's IP is not in the list of endpoints anymore. Another way to do it would be to get the YAML descriptor of the endpoints - which contains the names of the pods - and notice that our pod's name is not listed in there.

And while doing that, we can notice that our service still has the same number of IPs. Hum... let's check our running pods: yes, we have 1 more pod than before. That's because both the service and the deployment use the same label selector. So changing the label of the pod did also affect the deployment - or rather its replicaset - which in turn created a new pod to replace the one we took out of the pool.

So by using the same label selector for both, we're now able to quickly take a pod out of the pool. Which means that the pod won't be killed by the replicaset, and it won't receive traffic from the service. Which is very good when we want to debug a faulty application: you can just remove an instance from the production pool, knowing that another one will take its place automatically instead. And when you're done debugging, just delete the pod. It's as easy as that!

So let's remove our "debug" pod with:

```
$ kubectl delete pod -l debug
```

## Service Internals

But how does it work, internally? Kubernetes will watch for pods matching the configured label selector, and keep a set of iptable rules always up to date. These rules are defined on every nodes, and configured to redirect traffic destined to the service's virtual IP to a randomly selected pod - using round robin selection. And it's the same for node ports.

So it's not because you are requesting a specific node that your request will be handled by a pod on this node. The request might well be forwarded to a pod on a different node.

Another important thing to keep in mind, is that the services works at the transport level - layer 4 of the OSI model. Which is TCP/UDP in our case. So you won't be able to configure things like SSL, or do host or path-based matching. Those require working at the application level - layer 7 of the OSI model, which is HTTP. Using [Ingress](../ingress/README.md)!
