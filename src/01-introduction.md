# Introduction

Welcome!

This guide is the one I wish I had when I set out to provision my Kubernetes
cluster. It can be very difficult as a beginner to navigate the landscape where
everything is modular and there's several options for even the most basic
things like storage and networking, and everyone just says "it depends" with no
further context when asked which one to use. Sometimes, it's useful to just
have someone tell you what to use, so this is me, telling you what *I* use. You
may like it, you may not, but hopefully it can give you *something* to go on.

I recommend taking this slow. Build a lab with virtual machines or unused PCs
first and walk through this manual step by step. There is a lot involved in
setting up your first Kube cluster, so please do yourself a favor and don't
skip sections until you get to the maintenance guide. If there is information
you believe to be missing that can't be found easily online, feel free to open
an issue on the Github repo. I will only be covering the stack that I use. If
you'd like to use your own stack, feel free to fork this repo, but the point of
this is to give a set of good technologies that will fulfill the requirements
of most homelab environments, not to explore every option in detail. Once the
manual is complete, I would like to add a section that explores commonly-used
components and how they differ, but I will only be covering the installation
process for my chosen stack.


## Work in progress

Please note that this guide is not finished. I am actively writing it to
document how my new cluster is set up for disaster recovery purposes. There
will be many sections that just say "TODO". Please keep this in mind when
reading.


## Stack & Justification

TODO: comparisons

| Component | Chosen Technology | Required for basic operation |
|-----------|-------------------|----------|
| Operating System (OS) | Talos Linux | ✅️ |
| Container Runtime Interface (CRI) | Containerd | ✅️ |
| Container Network Interface (CNI) | Calico | ✅️ |
| Container Storage Interface (CSI) | Rook (Ceph) | ✅️ |
| Certificate management | CertManager | ❌️ (recommended) |
| Ingress / Gateway API controller | Traefik | ❌️ (recommended) |
| GitOps | ArgoCD | ❌️ (recommended) |
| Postgres databases | Cloud-Native Postgres (CNPG) | ❌️ |
| Virtual machine management | KubeVirt | ❌️ |


## Required Skills

Kubernetes is a beast, and should not be the first thing you go for when
learning about server administration or cloud environments. This guide assumes
you already have a solid foundation in the following areas:

- Git
- Linux System Administration
  - CLI
  - Disk management
  - Package management
  - Virtual machines
  - Certificate management (acme.sh, certbot, letsencrypt, or similar technologies)
- Linux Containers (one of Docker, Podman, etc)
- Networking Fundamentals
  - IP addressing
  - Subnetting
  - VLANs
  - Firewalls
  - Routing
