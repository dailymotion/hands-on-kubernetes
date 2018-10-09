# Kubernetes Local Setup

If you don't have access to a running Kubernetes cluster, you can easily start a local one, on your machine.

* [MacOS](#macos)
* [Linux](#linux)

### MacOS

Start by installing `kubectl`, the Kubernetes CLI client, using [Homebrew](https://brew.sh/):

```
$ brew install kubernetes-cli
```

And then, install Kubernetes. The easiest solution is to use [Docker for Mac](https://docs.docker.com/docker-for-mac/): it contains everything needed to run a local Kubernetes cluster. You can install it with [Homebrew Cask](http://caskroom.io/):

```
$ brew cask install docker
```

and follow the instructions on [Docker's documentation](https://docs.docker.com/docker-for-mac/#kubernetes) to enable Kubernetes (it's just a few clicks in the Docker UI). It will also automatically setup the right "credentials" for `kubectl` for your local cluster.

At the end, you should be able to run the following command:

```
$ kubectl cluster-info
```

and have something like the following response:

```
Kubernetes master is running at https://localhost:6443
KubeDNS is running at https://localhost:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

### Linux

If you are using Linux, the easiest solution is to use [Minikube](https://github.com/kubernetes/minikube), which "runs a single-node Kubernetes cluster inside a VM on your laptop".

You can follow the instructions on the [Minikube repository](https://github.com/kubernetes/minikube) to install it.

You will also need to [install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/). Note that minikube should have already set the right "credentials" file for `kubectl`, you should be able to run the following command:

```
$ kubectl cluster-info
```
