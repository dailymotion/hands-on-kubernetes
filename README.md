# Hands-on Kubernetes

Let's play with [Kubernetes](https://kubernetes.io/)!

The **goal** of this *hands-on* is to learn how to use Kubernetes to deploy applications. It goes through most of the concepts, and there are quite a few of them, so you will need some time to complete it. Let's have a look at what we will learn: 

- handling [namespaces](ns/README.md),
- deploying [pods](pod/README.md), [replicasets](rs/README.md) and [deployments](deployment/README.md),
- exposing pods through [services](svc-ep/README.md) and [ingress](ingress/README.md),
- managing [configuration](configmaps/README.md) and [secrets](secret/README.md),
- mounting [volumes](volume/README.md)

In order to follow this *hands-on*, you need an access to a running Kubernetes cluster:
- you can setup a [local Kubernetes cluster on your machine](local-setup/README.md)
- or use an already setup cluster - for example on GKE
- or even try some of the playground available on the internet, like [tryk8s.com](https://tryk8s.com/)

If you don't already have the `kubectl` client installed, it's time to [install it](https://kubernetes.io/docs/tasks/tools/install-kubectl/), and also maybe [configure it](https://kubernetes.io/docs/tasks/tools/install-kubectl/#configure-kubectl).

**To follow this *hands-on*, you will need `kubectl` >= 1.11**. You can check it by running `kubectl version`.

At this point, you should have a locally setup `kubectl` client. So running `kubectl cluster-info` should be working, and return information from the right cluster.

Before we continue, it would be a good idea to take a little time to [setup the shell autocompletion](https://kubernetes.io/docs/tasks/tools/install-kubectl/#enabling-shell-autocompletion).

I know some of you will not take the time to setup the autocompletion right now - I know you, you want to dive in right now! That's fine, but don't forget that you will use the `kubectl` client a lot, and it has a lot of commands and options, most of them dynamic (like a pod name), and a good completion will save you a lot of time and troubles! You've been warned ;-)

And if you have a little more time to spend on setup, you can also install a Kubernetes plugin/extension for your editor/IDE, for example:
- [Kubernetes extension for vscode](https://marketplace.visualstudio.com/items?itemName=ms-kubernetes-tools.vscode-kubernetes-tools)

You'll be writing quite a few YAML files, and having autocompletion / integrated documentation in your IDE will save you a lot of time!

One more thing before starting: if you find any mistake, typo, unclear sentence, missing information, or whatever else you'd like to fix, please feel free to do it. Open a PR, and improve this *hands-on* for the next people that will use it. Collaborative work!

Ok, so now that you are all setup, let's start by discovering how the `kubectl` client works, by running the following command:

```
$ kubectl
```

It will show you all the commands, grouped by type. For example, try the following command:

```
$ kubectl api-resources
```

It will show you all the "resources" that are supported by your Kubernetes cluster. That means all the operations you can do.

You can also have a look at the options of this command, with `kubectl api-resources -h`. A useful one is `kubectl api-resources -o wide`, which shows you more information (here, the actions you can perform on each resource kind) - and that we will use on other commands too.

All the commands share some common options, that you can get by running:

```
$ kubectl options
```

You can see for example the `-v` option, which controls the log level. The "level" here is an integer. The higher the value, the more verbose the output. For example, let's try the `api-resources` command, but with an increased log level, like:

```
$ kubectl api-resources -v 10
```

It will display a lot of things, and mainly for each API call to the Kubernetes master, the full request and response. Notice how for each API call, it will output a `curl` command, so that you can run the same call yourself (yes, there are lots of ideas/inspirations to take from Kubernetes!)

While we are playing with the basics, let's have a look at how (and where) the `kubectl` client stores its configuration:

```
$ kubectl config
```

It tells us that the configuration is stored in `~/.kube/config`. If you have a look at this file, you can see that it contains information about "clusters", "contexts" and "users":
- a **cluster** references a Kubernetes cluster
- a **user** contains "credentials" to connect to a cluster - usually an access token, or a client certificate
- a **context** is the association of a cluster and a user (if you need to connect to the same cluster with different credentials), along with an optional default namespace (more on namespaces later)

So if you need to work with multiple Kubernetes clusters, `kubectl` stores everything it needs to connect to them, but you can only use one at a time - referenced as "current context".

There are a few commands to make it easy to work with your local `kubectl` config:
- `kubectl config get-contexts` to see an overview of all the available contexts, and which one is the "default" one (= which will be used for all the commands)
- `kubectl config use-context` to switch between different contexts (note that if you need to run a single command against a different context, you can also use the `--context` option)

Another very useful resource for newcomers is the [kubectl cheatsheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/).

Now that we have the basics, like start working with [namespaces](ns/README.md)!
