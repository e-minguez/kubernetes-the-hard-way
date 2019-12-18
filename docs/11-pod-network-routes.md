# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network routes.

In this lab you will deploy [kube-router](https://github.com/cloudnativelabs/kube-router) to handle the network routes.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## Allow pod traffic in OpenStack

OpenStack will filter and drop all packets from IPs it does not know to prevent spoofing. This includes the pod and services CIDRs.

It is required to allow those IPs for traffic communication:

```
openstack port list --device-owner=compute:None -c ID -f value | xargs -tI@ openstack port set @ --allowed-address ip-address=10.200.0.0/16 --allowed-address ip-address=10.32.0.0/24
```

## Deploy kube-router

Each worker has the own pod subnet which was allocated by the Controller Manager. Then kube-route uses it to configure container networking.

> Alternatively, you can also specify the pod subnet for each node with a specific anotation:
> ```
>  kubectl annotate node/$(hostname -s) "kube-router.io/pod-cidr=<POD_CIDR>"
> ```
> The `POD_CIDR` should in the range of `cluster-cidr`, and no conflict with other pods.

Then, deploy `kube-router` pods:

```
DOMAIN="k8s.lan" \
CLUSTERCIDR=10.200.0.0/16 \
APISERVER="https://k8sosp.${DOMAIN}:6443" \
sh -c 'curl https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/generic-kuberouter-all-features.yaml -o - | \
sed -e "s;%APISERVER%;$APISERVER;g" -e "s;%CLUSTERCIDR%;$CLUSTERCIDR;g"' | \
kubectl apply -f -
```

> output

```
configmap/kube-router-cfg created
daemonset.apps/kube-router created
serviceaccount/kube-router created
clusterrole.rbac.authorization.k8s.io/kube-router created
clusterrolebinding.rbac.authorization.k8s.io/kube-router created
```

**IMPORTANT:** If for some reason you need to reinstall kube-router or change the config, check the `/var/lib/kube-router/kubeconfig` file on the workers. That's initialized when kube-router is installed, and stays there even if you remove it, so if you change addresses, make sure that file is up to date.

The nodes will be available now:

```
kubectl get nodes
```

> output

```
NAME               STATUS   ROLES    AGE     VERSION
worker-0.k8s.lan   Ready    <none>   3m55s   v1.15.3
worker-1.k8s.lan   Ready    <none>   3m55s   v1.15.3
worker-2.k8s.lan   Ready    <none>   3m55s   v1.15.3
```

Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
