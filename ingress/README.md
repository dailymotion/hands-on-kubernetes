# Kubernetes Ingress

* [Intro](#intro)
* [Install an Ingress Controller](#install-an-ingress-controller)
 * [Accessing the load balancer](#accessing-the-load-balancer)
* [Multiple services](#multiple-services)
* [Creating an ingress resource](#creating-an-ingress-resource)
* [Describing an ingress resource](#describing-an-ingress-resource)
* [Accessing an ingress resource](#accessing-an-ingress-resource)
* [Advanced configuration](#advanced-configuration)

## Intro

In the [services](../svc-ep/README.md) section, we learned how to access our application deployed in Kubernetes, both from inside the cluster, and from the outside world. But that was done using transport layer - at the TCP/UDP level. Which means you don't have access to all the features you might require from a modern LoadBalancer working at the application layer, such as TLS management, or host/path rules.

To use these kind of features, we'll use a more recent concept of Kubernetes, called [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/), that is an HTTP load balancer on top of one or more existing services.

There is a big difference between the ingress kind and the other kinds in Kubernetes: there is no controller for the ingress resources packaged in Kubernetes by default. This means that if you start creating a new ingress resource now... well, nothing will happens - except if your Kubernetes cluster already has an ingress controller setup, like the [GCE Load-Balancer Controller](https://github.com/kubernetes/ingress-gce) for GKE.

## Install an Ingress Controller

So if you are running on a local Kubernetes cluster, our first step will be to install an ingress controller. You can see a [list of available controllers](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-controllers). Here, we're going to use the [NGINX ingress controller](https://github.com/kubernetes/ingress-nginx).

We'll follow the following [installation guide](https://kubernetes.github.io/ingress-nginx/deploy/):

- first, install all the [mandatory resources](https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml) - if you click on the link, you will notice that a YAML file can contains multiple resources, which can be useful. This will create a new namespace `ingress-nginx` to host all these resources.
  ```
  $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
  ```
- make sure everything has been created, by listing all the resources from the `ingress-nginx` namespace - you should a deployment, with its replicaset and pod.
  ```
  $ kubectl get all -n ingress-nginx
  ```
- then, we'll follow the [Provider Specific Steps](https://kubernetes.github.io/ingress-nginx/deploy/#provider-specific-steps):
  - if you are on a Mac, using Kubernetes setup through [Docker for Mac](https://docs.docker.com/docker-for-mac/), you'll need to install the [cloud-generic resources](https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/cloud-generic.yaml):
    ```
    $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/cloud-generic.yaml
    ```
  - if you are using [Minikube](https://github.com/kubernetes/minikube):
    ```
    $ minikube addons enable ingress
    ```

The important part here is that everything was installed in the `ingress-nginx` namespace. So it won't affect your work in the `hands-on` namespace, and if at the end of the hands-on, you want to remove the [NGINX ingress controller](https://github.com/kubernetes/ingress-nginx), you will just have to delete its namespace.

### Accessing the load balancer

At this point, we have a local nginx instance, which is exposed through a `LoadBalancer` service. So what we did is just to create an indirection: we now have a extra layer between the external world and our pods: an incoming HTTP request will now:
- enter through the `ingress-nginx` service
- pass through a nginx pod, where it will be directed to a destination service
- goes through the destination service - for example a `kubernetes-up-and-running` service
- and finally reach a destination pod - for example a `kubernetes-up-and-running` pod

This workflow allows more control over the routing configuration, because it can now be defined as a Kubernetes resource, instead of being defined in an external load-balancer.

Even if we don't have any ingress resource defined, we can still test our nginx instance. We just need to have a look at its service definition:

```
$ kubectl describe svc ingress-nginx -n ingress-nginx
```

2 side notes before commenting the service itself:
- we're using the `-n` flag to get a resource from a different namespace
- if you want to have the autocompletion when working with a different namespace, you should write the `-n` flag before the resource name, for example with: `kubectl describe pod -n ingress-nginx <tab>`

So, our service is exposed both on port `80` and `443`. Let's try it:
- <http://localhost/>
- <https://localhost/>

If you get a 404 served by nginx, it means that everything is working! It's returning a 404 because we don't have an ingress resource yet...

## Multiple services

So, now that the plumbing stuff is done, let's get back to our work. Previously, we deployed a single version of an application, and enable access to it through a service. Let's imagine now that we would like to have 2 different versions of an application running at the same time, for example because we still want to support old clients using the old version, while allowing new clients to benefits from the new version. That would require:
- 2 different deployments, 1 for each version
- 2 different services, 1 for each deployment

But then, instead of having to setup 2 different DNS, we would like to use a single one, and a path prefix for each version:
- `/v1` for version 1
- `/v2` for version 2

so if later we want to deploy version 3, we can easily expose it under `/v3`.

To do that, we'll use a single ingress resource, that will forward all traffic for urls starting with `/v1` to the first service, and all traffic for urls starting with `/v2` to the second service.

We won't use the [kubernetes up and running](https://github.com/kubernetes-up-and-running/kuard) application here, because it's a pain to expose it through a different path, and handle rewriting. So instead we'll use a simpler application here: the [hello-app](https://github.com/GoogleCloudPlatform/kubernetes-engine-samples/tree/master/hello-app).

So we'll need to create our deployments and services:
- a `hello-app-v1` deployment using the [gcr.io/google-samples/hello-app:1.0](https://gcr.io/google-samples/hello-app:1.0) docker image, and an associated `hello-app-v1` service
- a `hello-app-v2` deployment using the [gcr.io/google-samples/hello-app:2.0](https://gcr.io/google-samples/hello-app:2.0) docker image, and an associated `hello-app-v2` service

To do it quickly, we'll use the `kubectl create deployment` command, that can create a deployment with a single container:

```
$ kubectl create deployment hello-app-v1 --image=gcr.io/google-samples/hello-app:1.0
$ kubectl create deployment hello-app-v2 --image=gcr.io/google-samples/hello-app:2.0
```

and then the `kubectl expose` to expose our deployments through services:

```
$ kubectl expose deployment hello-app-v1 --port 8080
$ kubectl expose deployment hello-app-v2 --port 8080
```

We are using the default `ClusterIP` type of service, because our service won't be exposed directly to the outside world - it will be accessed by our nginx "load balancer" instance, which is running inside the cluster.

## Creating an ingress resource

Now that the plumbing stuff is done - again - we can start creating our ingress resource. We will write the descriptor file, based on the [ingress definition in the API reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#ingress-v1beta1-extensions). Take care that the `Ingress` kind is part of the `extensions` group, with the `v1beta1` version.

The solution can be found at [hello-app-ingress.yml](hello-app-ingress.yml). You can see that we are defining 2 "backends", 1 for each path. We can then create it, using `kubectl apply`:

```
$ kubectl apply -f hello-app-ingress.yml
```

## Describing an ingress resource

As with any other resource, we can get or describe an ingress resource:

```
$ kubectl describe ingress hello-app
```

we can see for each path, the destination backend: service name and port.

## Accessing an ingress resource

So let's try it:
- <https://localhost/v1>
- <https://localhost/v2>

We can see that our request to `/v1` was forwarded to the `-v1` service, because the response is coming from a pod running version 1.

## Advanced configuration

The main advantage of running the load-balancer inside the cluster, is to be able to configure it using Kubernetes tools. In our case, we are using NGINX, and its [configuration](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/) can be expressed either:
- in the ingress resource's annotations
- in config maps - we'll have a look at it in the next section

This allows you to configure things like:
- enforcing https
- timeouts
- CORS
- rate limiting
- redirects
- ...

Of course this is specific to the ingress controller's implementation. Other implementations will have different configuration options, or even be use load-balancers outside of the cluster - for example on GKE.
