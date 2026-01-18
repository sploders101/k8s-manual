# Initial Bringup

I'll be using Talos Linux, because having provisioned my own cluster with
kubeadm, it's what I wish I used to begin with. This section goes over the
bring-up process for Talos Linux using my chosen Kubernetes stack.


## Generate base configs

TODO


## Get an install image

TODO


## Start nodes

Talos seems to be massively overcomplicated for network configuration. It's
probably best to stick to DHCP with static leases for now...

First, bring up the nodes with appropriate install images. Once you see the
linux logs, you can remove the drive and move on to the next node.

Once the node is booted, you can get its mac address with the following
command:

```bash
talosctl get links --insecure --nodes <node_ip>
```

This may help with DHCP static lease configuration. Not amazing that it has to
be done *after* it gets a lease already, but whatever... It would be nice if
they displayed the MAC ***ANYWHERE*** on the dashboard in maintenance mode...


## Create patch files

Next, you'll want to create a patch file for each node. This provides important
information to the installer that may or may not be specific to that node, such
as the data disk, the system schematic (for adding extra drivers, etc), and
overrides for some default components.

### Disks

Use the following command to list disks on the node:

```bash
talosctl get disks --insecure --nodes <node_ip>
```

### Schematic

Use the image factory at the following link to acquire an "Initial Instalation"
image URL:

[Talos Image Factory](https://factory.talos.dev/)

### CNI Override

Also, if you don't like Flannel and want to use a CNI capable of actually
isolating pods/namespaces and controlling traffic, make sure to include
`cluster.network.cni.name = false`, as shown in my example below. Flannel is
great for a demonstration, but doesn't include any kind of network policy
management, so you may want to use something like Calico instead.

### Example

Use this information to create a configuration file with a `.yaml` extension
similar to this one:

```yaml
machine:
  install:
    disk: /dev/vda
    image: factory.talos.dev/metal-installer-secureboot/d65015d8cb6aeafd3607f403cf96d63c5e1d9d16cda42709dc42c5c1e85f1929:v1.12.1
cluster:
  network:
    cni:
      name: none
```


## Add pull-through cache mirrors (optional)

If you have a pull-through cache set up (most probably to mitigate docker.io's
rate limiting errors), you can add the following config to your patch files to
ensure each node is set up to use your cache (duplicate for each registry):

```yaml
---
apiVersion: v1alpha1
kind: RegistryMirrorConfig
name: docker.io
endpoints:
    - url: https://<your_domain>/v2/<cache_namespace>
      overridePath: true
```

This can be used with JFrog Artifactory or Harbor's pull-through caches, or the
cache_namespace piece can be removed if you're using a cache that doesn't serve
them under a subdirectory.


## Apply config to nodes

Next, you'll want to apply your configurations to the nodes. You can do this
using the following command:

```bash
talosctl apply-config --insecure --file <base_config> --config-patch <node_patch_file> --nodes <node_ip>
```

Example base_configs are: `controlplane.yaml`, `worker.yaml`. The
node_patch_files are the patch files you created in the previous step. I
recommend having one for each node.


## Configure the kubernetes endpoint & bootstrap

Next, you'll want to configure the kubernetes endpoint. This can be an IP
address, a series of IP addresses, or a DNS name resolving to one or more IP
addresses or load balancers. If you have the ability to set DNS records on your
home network, I recommend this approach. I'm not sure how well Talos does with
reconfiguration of this value, but the kubeadm docs say it cannot be changed.
Using DNS will allow you to edit this value later.

```bash
talosctl config endpoint <control_plane_name_or_ip ...>
```

Once that's configured, you can bootstrap the cluster using the following
command:

```bash
talosctl bootstrap --nodes <control_plane_ip>
```

This will prompt Talos to set up etcd and bring up the cluster.


## Add cluster to kubectl contexts & monitor cluster bring-up

Next, you'll want to access the Kubernetes API of the cluster and check on the
progress of cluster bring-up. Use the following command to add the Talos
cluster to your kubectl contexts:

```bash
talosctl config merge ./talosconfig
```

Now, you can use the following command to see the nodes in the cluster:

```bash
kubectl get nodes
```

If you don't see all of your nodes (including the control plane), try again.
It may take a little bit for them all to appear.


## Add CNI (if not using Flannel)

This step is only required if you opted out of using Flannel CNI. I will
document the process for adding Calico, but if you choose to use something
else, you can ignore this section and follow the instructions for the CNI you
chose.

Apply the tigera operator manifest:

```bash
kubectl create -f https://docs.tigera.io/calico/latest/manifests/tigera-operator.yaml
```

Next, you'll need a configuration for the operator to work with. Assuming this
doesn't conflict with the IP address scheme you already have, this config will
get you up and running quick & easy:

```yaml
---
# This section includes base Calico installation configuration.
# For more information, see: https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.Installation
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    ipPools:
    - name: default-ipv4-ippool
      blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
---
# This section configures the Calico API server.
# For more information, see: https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.APIServer
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
```

## Conclusion

Congratulations! You have a cluster! Here's a brief summary of what we just did:

- Download Talos Linux and flash it to a USB drive
- Boot Talos Linux on all nodes
- Generate certificates for the Talos & Kubernetes API servers
- Write patch files for each node based on information we retrieved from the CLI
- Install Talos into the nodes
- Configure kubectl to control the Talos cluster
- Install Calico CNI
- Of the 4 main requirements for Kubernetes, we have installed:
  - OS
  - CRI
  - CNI
