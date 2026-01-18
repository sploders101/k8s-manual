# Introduction

Welcome!

This guide is the one I wish I had when I set out to provision my Kubernetes
cluster. It can be very difficult as a beginner to navigate the landscape where
everything is modular and there's several options for even the most basic
things like storage and networking, and everyone just says "it depends" with no
further context when asked which one to use. Sometimes, it's useful to just
have someone tell you what to use, so this is me, telling you what *I* use. You
may like it, you may not, but hopefully it can give you *something* to go on.

## Work in progress

Please note that this guide is not finished. I am actively writing it to
document how my new cluster is set up for disaster recovery purposes. There
will be many sections that just say "TODO". Please keep this in mind when
reading.

## Stack & Justification

TODO: comparisons

* OS: Talos Linux
* CRI: containerd
* CNI: calico
* CSI: rook/ceph
