# Networking Setup

For networking, we will be using Calico for our CNI and MetalLB for our
`LoadBalancer` service manager.

## Calico Setup

First, we'll set up our CNI. If you've opted to use Flannel, you can skip this
section. Otherwise, go ahead and install the Tigera Operator using Helm.

First, create your `values.yml` file for Tigera Operator:

```yaml
installation:
  cni:
    type: Calico
  calicoNetwork:
    bgp: Disabled
    ipPools:
      - cidr: 10.244.0.0/16
        encapsulation: VXLAN
```

Next, install the operator:

```bash
helm repo add projectcalico https://docs.tigera.io/calico/charts
helm install \
  --create-namespace \
  --namespace tigera-operator \
  --version v3.31.3 \
  -f values.yml \
  calico \
  projectcalico/tigera-operator
```

This will configure Calico to use `10.244.0.0/16` as range to use for IPAM (IP
address management), and use VXLAN, which is a type of overlay network that
allows the pod network to safely cross your existing network when communicating
between nodes.

## MetalLB Setup

Finally, we'll set up MetalLB. This will provide the `LoadBalancer` service
implementation.

You'll need:
- The names of the interfaces to use for advertising address changes
- A range of IP addresses that MetalLB can take exclusive control of

First, create the metallb-system namespace with extra privileges:

```yaml
# kubectl apply -f <file>
---
apiVersion: v1
kind: Namespace
metadata:
  name: metallb-system
  labels:
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/warn: privileged
```

Then install MetalLB with Helm:

```bash
helm repo add metallb https://metallb.github.io/metallb
helm install --namespace metallb-system metallb metallb/metallb
```

And finally, add your configuration. Make sure to add your network interfaces
and IP pools:

```yaml
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: simple-services-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - simple-services
  interfaces:
  - eno1
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: simple-services
  namespace: metallb-system
spec:
  addresses:
  - x.x.x.x-x.x.x.x
```
