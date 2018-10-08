# Kubernetes Namespaces

A [namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) is used to group multiple resources together, and also avoid naming collisions: every created object needs to have a unique name - inside the namespace. Each Kubernetes cluster has a `default` namespace, but today we'll work in a specific namespace.

So let's start by creating a new namespace:

```
$ kubectl create namespace hands-on
```

Great! Now we have our new namespace. We can make sure it worked by listing all namespaces, with the `kubectl get` command:

```
$ kubectl get namespaces
```

Notice that you can use either `kubectl get namespace` or `kubectl get namespaces`, both will work. And if you are lazy, you can even use `kubectl get ns`, which is the shortname for namespace (you can see all supported shortnames in `kubectl api-resources`).

Now, we need to update our current context to run every command in this new namespace by default:

```
$ kubectl config set-context $(kubectl config current-context) --namespace=hands-on
```

This is the recommanded way to update your `kubectl` config (avoid manually editing the configuration file - use the `kubectl config` command instead).

So if you run the `kubectl config get-contexts` now, you will that our current context has a namespace set.

Note that even if by default, the `hands-on` namespace will be use, you can always force a specific namespace for a command, by using the `--namespace` flag.

At the end of the hands-on, if you want to delete everything, you can run:

```
$ kubectl delete namespace hands-on
```

and it will also delete all resources created in this namespace.

But before deleting our namespace, let's put some resources in it, starting with [some pods](../pod/README.md)!
