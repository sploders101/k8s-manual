# Rook Ceph

Rook is a management overlay for Ceph that can deploy it in a Kubernetes
cluster. If you don't know what Ceph is, I highly recommend watching some of
the videos on it by the guys at 45drives.

- [45drives Ceph Series on YouTube](https://www.youtube.com/watch?v=i5PIFeWPpHM&list=PL0q6mglL88AP4ssmkDn_mMkozuQiLr2dv)

The elevator pitch: "It's like RAID, but it joins multiple servers instead of
just drives."

The more nuanced explanation:
- Imagine if you could take every piece of data you want to store and break it
  down into manageable byte-sized pieces.
- Now imagine that you could replicate each of these pieces of data a number of
  times to create redundancy, so if you lose one of them, you have replicas to
  pull from, and when all is functioning normally, you could run periodic
  checks to make sure that all of your replicas actually have the same data. If
  two replicas say the data is "a", but one says it's "b", then chances are
  that "b" was supposed to be an "a". This solves both availability and
  integrity problems (like bitrot).
- Now imagine that you could build an algorithm that takes into account the
  number of drives you have, the size of each one, the number of hosts you
  have, which drives were in which hosts, as well as your entire datacenter
  heirarchy, and use that to determine to which drive a piece of data should go
  to maintain a certain amount of redundancy across any failure domain of your
  choice.
- Now imagine that each drive had its own server that you could communicate
  with directly, and there was a server to ensure that not only are your
  redundancy requirements enforced when the data is created, but also as
  requirements and resource availability change.
- Now imagine that you could build interfaces on top of this data storage
  strategy which exposes a filesystem, a block device, and an S3-compatible
  object store.

That's Ceph.

It's big, and it's complicated, but it is a true marvel of engineering that
solves an extremely complicated problem in a way that *any* application can
interface with and have no idea that anything has changed. One of my personal
biggest use cases for it is CephFS. CephFS is the only open source network
filesystem that I know of which is fully POSIX compliant, and it's built on a
rock-solid data redundancy system that has never fully failed on me except when
I got paranoid and forced it to do something stupid.

I highly recommend learning to manage Ceph properly, but what's even nicer
about all this is that unless you're doing anything crazy like me, you likely
won't have to do anything more than initiate automated repair for occasional
data corruption after heavy use. I'll get into some more Ceph management
essentials in a later section of the book, but Rook takes care of most of it
for you.

Side note: If Ceph tells you that your PGs are inconsistent, don't freak out.
It knows what it's doing, and if you instruct it to repair, it almost always
will, but it takes some time, so stay calm and give it space to work. I'll go
over repairing data in the maintenance guide.


## Installation

We will be using Helm to install Rook. I am writing this manual off of the
official docs which can be found here:

[Ceph Operator Helm Chart](https://rook.io/docs/rook/v1.10/Helm-Charts/operator-chart/#installing)

### \[Aside\] Why Helm?

I recommend using the Helm repo instead of applying the manifest files directly
because helm can help you manage resources after they've been deployed. This is
greatly helpful for things like upgrades where resources may be added,
modified, or removed, and can be difficult to keep track of by hand.

Normally, I would recommend tracking everything in a GitOps platform like
FluxCD, but since Ceph provides the CSI that FluxCD needs in order to run
properly, we unfortunately cannot use FluxCD yet.

### Install Rook Ceph Operator

First, add the Rook Helm chart repository.

```bash
helm repo add rook-release https://charts.rook.io/release
```

Next, create your `values.yml` file. I personally like the default config, so
I won't be creating one. However, you can use
[this section of the docs](https://rook.io/docs/rook/v1.10/Helm-Charts/operator-chart/#configuration)
to determine the variables you want to use. Note that the `.` characters used
in the parameter names represent nested values. For example, if you want to set
the `crds.enabled` parameter to `false`, you'd use the following yaml:

```yaml
crds:
  enabled: false
```

If you choose to customize any values, make sure to add `-f values.yml` to the
end of your install command.

According to [the Talos Linux documentation](https://docs.siderolabs.com/kubernetes-guides/csi/ceph-with-rook#installation),
the default configuration does not allow privileged pods. You'll need to create
the namespace first to allow them:

```yaml
# kubectl apply -f <file>
---
apiVersion: v1
kind: Namespace
metadata:
  name: rook-ceph
  labels:
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/warn: privileged
```

Ceph needs privileged pods to effectively manage disks on the host.

Once that's done, install the Rook Ceph Operator Helm chart, use the following
command:

```bash
helm install --namespace rook-ceph rook-ceph rook-release/rook-ceph
```

### Install Rook Ceph Cluster

Now that the Rook Ceph operator has been installed, we need to add a
`CephCluster` resource in order to get it to provision a Ceph Cluster. This is
because the operator is not itself a cluster. It is an automated interface for
*managing* clusters, and is actually capable of managing more than one.
Therefore, we need to give it a cluster definition for it to build one. You can
find details on this process at the following link:

[CephCluster CRD Documentation](https://rook.io/docs/rook/v1.10/CRDs/Cluster/ceph-cluster-crd/)

[CephCluster Full Example](https://github.com/rook/rook/blob/master/deploy/examples/cluster.yaml)

I'll be creating a cluster that spans the whole cluster and consumes all
unpartitioned disks.

```yaml
# kubectl apply -f <file>
---
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v19.2.3
  dataDirHostPath: /var/lib/rook
  mon:
    count: 3
    allowMultiplePerNode: false
  mgr:
    count: 1
    allowMultiplePerNode: false
    modules:
      # List of modules to optionally enable or disable.
      # Note the "dashboard" and "monitoring" modules are already configured by
      # other settings in the cluster CR.
      # I recommend the "rook" module to inform the dashboard that Ceph
      # resources are configured by Kubernetes manifests.
      # I also recommend the "nfs" module as it will provide easy configuration
      # of NFS exports via the dashboard. This is very useful for things like
      # pre-seeding PVCs, data export, and troubleshooting.
      - name: rook
        enabled: true
      - name: nfs
        enabled: true
  dashboard:
    enabled: true
    ssl: true
  storage:
    useAllNodes: true
    useAllDevices: true
    config:
      encryptedDevice: "true"
  disruptionManagement:
    managePodBudgets: true
  # This can be turned on to help with OSD removal. Since OSDs will be
  # automatically marked "out" if they are offline for too long, I recommend
  # keeping this off except when you need it.
  removeOSDsIfOutAndSafeToRemove: false
```

### Install rook-ceph Krew Plugin

Much of Ceph's administration post-install happens via CLI, so you'll want to
make sure you have it. You can either deploy the "toolbox" container (the
official docs go over this), or you can use the kubectl plugin. I recommend the
plugin for simplicity. Assuming you installed `krew` from the tools section,
you can get the rook-ceph plugin using the following command:

```bash
kubectl krew install rook-ceph
```

Now you can run ceph commands like so:

```bash
# `ceph status` becomes...
kubectl rook-ceph ceph status
```

I recommend adding aliases to the rc file for whatever shell you use:

```bash
alias ceph='kubectl rook-ceph ceph'
alias rbd='kubectl rook-ceph rbd'
alias rados='kubectl rook-ceph rados'
alias radosgw-admin='kubectl rook-ceph radosgw-admin'
```

This will cover most commands you might find in the official Ceph
documentation. Ordinarily, you'd run these commands from a Ceph host, but since
Rook is provisioning everything for us and we don't necessarily have direct
access, it's a lot easier to use the plugin. This will run your commands in the
"operator" container and attach stdio as if it were running locally.

### Add CephFS Filesystem

CephFS is the interface you're primarily going to want to use for a Homelab
since it provides the greatest degree of flexibility if all you need is a POSIX
filesystem for application data.

It provides support for the `ReadWriteMany` and `ReadOnlyMany` PVC modes, which
is extremely useful for things like Plex/Emby/Jellyfin libraries and other file
shares, as well as quasi, mostly-stateless things like the Home Assistant voice
pipeline components, which, apart from caching models, don't share any state
and could easily be replicated.

#### Prior Considerations

CephFS is very flexible and can be configured in a number of ways. Personally,
due to certain operational requirements (and the simplicity of doing so), I
like to keep all of my storage mechanisms in the same CephFS filesystem. I'll
get to why this is important in a moment.

Rook supports 2 main device classes, and auto-detects which one each OSD
belongs to during OSD preparation. These classes are `ssd` and `hdd`. CephFS
supports storing data on devices of multiple classes, but ***THESE MUST BE
DETERMINED UPFRONT***! If you fail to specify your device classes upfront,
you'll have to resort to manual editing of your CRUSH map, and even then, the
Ceph operator will have no idea what's going on and be extremely annoying to
deal with. Alternatively, you could just deal with your data being striped
across both SSDs and HDDs, but this will likely yield a weird and sub-par
experience. In summary, ***PLEASE*** specify your storage classes, even if you
only have one right now. You'll thank me later...

I recommend always storing metadata on ssd storage classes. This is a heavy
random I/O operation, which is what SSDs were designed for, as opposed to
spinning disk drives which are better for bulk storage and sequential I/O. If
you have enough space, I also recommend making this your default storage medium
altogether, since you'll usually have more control over the provisioning of
mass storage volumes than database volumes, and databases do a lot of random
I/O.

#### CephFS Definition

Here is the definition I use. I use ec-2-1 for massive data that I'd like to be
highly available, but could be taken down and restored from a backup if it
needs to be. I then have two data pools: one for ssd, and one for hdd with 3
replicas for normal storage.

```yaml
# kubectl apply -f <file>
---
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: kubefs
  namespace: rook-ceph # namespace:cluster
spec:
  # The metadata pool spec
  metadataPool:
    deviceClass: ssd
    replicated:
      # You need at least three OSDs on different nodes for this config to work
      size: 3
  # The list of data pool specs
  dataPools:
    - name: replicate-3-ssd
      deviceClass: ssd
      replicated:
        size: 3
~    - name: replicate-3-hdd
~      deviceClass: hdd
~      replicated:
~        size: 3
~    # You need at least three OSDs on different nodes for this config to work
~    - name: ec-2-1-hdd
~      deviceClass: hdd
~      erasureCoded:
~        dataChunks: 2
~        codingChunks: 1
~      parameters:
~        compression_mode: none
  # Whether to preserve filesystem after CephFilesystem CRD deletion
  preserveFilesystemOnDelete: true
  metadataServer:
    activeCount: 1
    activeStandby: true
    # The affinity rules to apply to the mds deployment
    placement:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
                - key: filesystem
                  operator: In
                  values:
                    - media
            topologyKey: kubernetes.io/hostname
        preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - rook-ceph-mds
              # topologyKey: */zone can be used to spread MDS across different AZ
              topologyKey: topology.kubernetes.io/zone
    annotations:
    labels:
      filesystem: kubefs
    resources:
```

### Add Storage Classes

Finally, you'll need to add a storage class for each pool you want to use for
PVC provisioning. For my configuration above, I add the following storage
classes (all but one hidden for brevity):

```yaml
# kubectl apply -f <file>
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-kubefs-replicate-3-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rook-ceph.cephfs.csi.ceph.com # csi-provisioner-name
parameters:
  # matches the name of the CephCluster resource
  clusterID: rook-ceph
  # matches the name of the CephFilesystem resource
  fsName: kubefs
  # matches the name of a dataPool object within the CephFilesystem resource
  pool: kubefs-replicate-3-ssd

  # The secrets contain Ceph admin credentials. These are generated automatically by the operator
  # in the same namespace as the cluster.
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph

  mounter: kernel
reclaimPolicy: Delete
allowVolumeExpansion: true
~---
~apiVersion: storage.k8s.io/v1
~kind: StorageClass
~metadata:
~  name: rook-kubefs-replicate-3-hdd
~provisioner: rook-ceph.cephfs.csi.ceph.com
~parameters:
~  clusterID: rook-ceph
~  fsName: kubefs
~  pool: kubefs-replicate-3-hdd
~
~  # The secrets contain Ceph admin credentials. These are generated automatically by the operator
~  # in the same namespace as the cluster.
~  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
~  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
~  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
~  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
~  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
~  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
~
~  mounter: kernel
~reclaimPolicy: Delete
~allowVolumeExpansion: true
~---
~apiVersion: storage.k8s.io/v1
~kind: StorageClass
~metadata:
~  name: rook-kubefs-ec-2-1-hdd
~provisioner: rook-ceph.cephfs.csi.ceph.com # csi-provisioner-name
~parameters:
~  clusterID: rook-ceph
~  fsName: kubefs
~  pool: kubefs-ec-2-1-hdd
~
~  # The secrets contain Ceph admin credentials. These are generated automatically by the operator
~  # in the same namespace as the cluster.
~  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
~  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
~  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
~  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
~  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
~  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
~
~  mounter: kernel
~reclaimPolicy: Delete
~allowVolumeExpansion: true
```

# Testing the Filesystem

To test your brand new filesystem, create a `PersistentVolumeClaim` and see
what happens:

```yaml
# kubectl apply -f <file>
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc-test
  namespace: default
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

Then look at it using this command:

```bash
kubectl get pvcs
```

If you see something like this with status=Bound:

```
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                  VOLUMEATTRIBUTESCLASS   AGE
cephfs-pvc-test   Bound    pvc-e2a72895-e95f-4a69-a042-ecfe9480c5aa   1Gi        RWX            rook-kubefs-replicate-3-ssd   <unset>                 4s
```

Congratulations! You've successfully set up CephFS as your container storage
interface! Now, in the next session, you'll learn (among other things) how to
set up an NFS server to access this new volume. If you decide not to set up
NFS, you can go ahead and delete the PVC with
`kubectl delete pvc cephfs-test-pvc`. Otherwise, continue to the next section.
