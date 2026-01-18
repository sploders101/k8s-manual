# Helm

For some of our next tasks, we're going to be using a Kubernetes package manager
called "Helm". Helm is to Kubernetes what apt is to Debian Linux. It collects
sets of resources, allows customization, and keeps track of them, allowing you
to easily add, remove, and update without having to worry about things like
cleaning up garbage from old package versions.

We won't be interacting with Helm directly much in this guide - just for
installing the absolute essentials - because there's another tool called ArgoCD
that will allow you to track your manifests, Helm charts, etc from a git
repository.


## Basic Concepts

Helm actually has pretty good documentation, so I won't go into much detail
here. The one thing you really shouldn't miss is that customizations to a Helm
chart are done using a `values.yaml` file. Each chart defines its own that it
uses within a series of template files that pull in your values to generate a
Kubernetes manifest (collection of resources defined in yaml).

- [Using Helm](https://helm.sh/docs/intro/using_helm)
- [Cheat Sheet](https://helm.sh/docs/intro/CheatSheet)


## Installing Helm

Helm has instructions for installation at the following link:

- [Install Helm](https://helm.sh/docs/intro/install/)


## Using Helm

Helm has a concept of "repos", similar to apt, which allow people to host their
own collections of kubernetes manifests as Helm charts. The general process for
installing a helm chart is as follows:

```bash
helm repo add <repo_name> <url>
helm get values <repo_name>/<chart_name> -o values.yml
helm install --namespace <namespace_name> <arbitrary_name> <repo_name>/<chart_name> -f values.yaml
```
