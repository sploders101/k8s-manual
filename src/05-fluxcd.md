# FluxCD (TODO)

FluxCD isn't required for a Kubernetes cluster, but I strongly recommend it. I
haven't personally used this particular offering before (my experience is with
ArgoCD), so this section may have some rough edges until the manual is
finished.

I have chosen to use FluxCD for my cluster rebuild project because of its
low-dependency operating style. It seems to follow the UNIX philosophy of doing
one thing and doing it well. It has no UI, no auth system, just a
synchronization controller that ensures your Kube cluster is synced with your
Git project. After installing it, I realized it doesn't even have any PVCs,
which means it can be used for your Rook Ceph installation as well.

To install FluxCD, pick a provider for your Git repository and follow their
documentation linked below:

[FluxCD Bootstrap Guide](https://fluxcd.io/flux/installation/bootstrap/)

FluxCD also has a "Getting Started" guide that will take you through some more
details past bootstrap:

[FluxCD Getting Started](https://fluxcd.io/flux/installation/bootstrap/)

The rest of this guide will assume you've gone through FluxCD's getting started
guide and have at least done the exercise.


## Overview

FluxCD adds custom resource definitions (CRDs) to Kubernetes and manages
deployment of resourced based on these CRDs. It regularly polls data sources,
checks for updates, and redeploys them automatically.

NOTE: The examples here are to give you a basic understanding of how FluxCD
deploys resources. They are not intended for you to follow along.

### 


## Managing Helm Charts


