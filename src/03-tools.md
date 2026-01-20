# Tools

Managing a Kubernetes cluster can be complicated and difficult. Fortunately,
Kubernetes is all about automation, and we have a variety of tools at our
disposal to help with all this. In this section, we will briefly touch on a few
of them that get used throughout this guide. There may be more tools in other
sections that are specific to those sections, but this page will walk you
through some of the common ones you'll use frequently.


## Helm

First up is Helm. Helm is to Kubernetes what apt is to Debian Linux. It
collects sets of resources, allows customization, and keeps track of them,
allowing you to easily add, remove, and update without having to worry about
things like cleaning up garbage from old package versions.

We won't be interacting with Helm directly much in this guide - just for
installing the CNI and for rendering templates - because there's another tool
called FluxCD that will allow you to track your manifests, Helm charts, etc
from a git repository.


### Basic Concepts

Helm actually has pretty good documentation, so I won't go into much detail
here. The one thing you really shouldn't miss is that customizations to a Helm
chart are done using a `values.yaml` file. Each chart defines its own that it
uses within a series of template files that pull in your values to generate a
Kubernetes manifest (collection of resources defined in yaml).

- [Using Helm](https://helm.sh/docs/intro/using_helm)
- [Cheat Sheet](https://helm.sh/docs/intro/CheatSheet)


### Installing Helm

Helm has instructions for installation at the following link:

- [Install Helm](https://helm.sh/docs/intro/install/)


### Using Helm

Helm has a concept of "repos", similar to apt, which allow people to host their
own collections of kubernetes manifests as Helm charts. The general process for
installing a helm chart is as follows:

```bash
helm repo add <repo_name> <url>
helm get values <repo_name>/<chart_name> -o values.yml
helm install --namespace <namespace_name> <arbitrary_name> <repo_name>/<chart_name> -f values.yaml
```


## Krew

Next, we have Krew. Krew is a "plugin manager" for kubectl. Some frameworks
that you install in Kubernetes have a lot going on, or wrap some kind of
pre-existing technology and require some extra hoops to interacting with them.

This is where kubectl plugins come in. They are a way to extend the kubectl CLI
with additional functionality. For example, in a later section, you'll be
installing the `rook-ceph` kubectl plugin through krew to interact with the
Ceph CLI via Kubernetes. In another section, you'll be installing the `virt`
kubectl plugin to interact with KubeVirt for managing your virtual machines.
Krew is a handy package manager for installing these plugins.


### Installation

Krew actually has an installation guide that's short and sweet, so you can just
follow the instructions here:

[Krew Installation Guide](https://krew.sigs.k8s.io/docs/user-guide/setup/install/)


### Using Krew

[Official Quickstart Guide](https://krew.sigs.k8s.io/docs/user-guide/quickstart/)

Krew is pretty easy to use, and usually when you need it, the documentation of
whatever you're working on will tell you how to use it. Here's a brief overview
though just in case.

Update package cache:
```bash
kubectl krew update
```

Search for a package:
```bash
kubectl krew search <keyword>
```

Install a package:
```bash
kubectl krew install <package>
```

Upgrade packages:
```bash
kubectl krew upgrade
```

Uninstall a package:
```bash
kubectl krew uninstall <package>
```
