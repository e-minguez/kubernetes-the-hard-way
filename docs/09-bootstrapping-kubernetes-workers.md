# Bootstrapping the Kubernetes Worker Nodes

In this lab you will bootstrap three Kubernetes worker nodes. The following components will be installed on each node: [runc](https://github.com/opencontainers/runc), [container networking plugins](https://github.com/containernetworking/cni), [containerd](https://github.com/containerd/containerd) & [kubelet](https://kubernetes.io/docs/admin/kubelet).

**For [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies) we will use `kube-router` instead.**

## Prerequisites

Fix the workers hostnames just in case:

```
for instance in worker-0 worker-1 worker-2; do
  ssh -i ~/.ssh/k8s.pem centos@${instance}.${DOMAIN} sudo hostnamectl set-hostname ${instance}.${DOMAIN}
  ssh -i ~/.ssh/k8s.pem centos@${instance}.${DOMAIN} hostname
done
```

The commands in this lab must be run on each worker instance: `worker-0`, `worker-1`, and `worker-2`. Login to each worker instance:

```
ssh -i ~/.ssh/k8s.pem centos@worker-0.${DOMAIN}
```

### Running commands in parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. See the [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux) section in the Prerequisites lab.

## Provisioning a Kubernetes Worker Node

Install the OS dependencies:

```
{
  sudo yum install epel-release -y
  sudo yum install socat conntrack ipset wget jq -y
  sudo yum-config-manager --disable epel
}
```

> The socat binary enables support for the `kubectl port-forward` command.

Disable selinux (I know, I know):

```
sudo sed -i -e 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
```

Update to the latest version of all the packages and reboot the instances just in case:

```
{
  sudo yum update -y
  sudo reboot
}
```

### Download and Install Worker Binaries

Login back to the workers and perform the following commands:

```
wget -q --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.15.0/crictl-v1.15.0-linux-amd64.tar.gz \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc8/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v0.8.2/cni-plugins-linux-amd64-v0.8.2.tgz \
  https://github.com/containerd/containerd/releases/download/v1.2.9/containerd-1.2.9.linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kubelet
```

Create the installation directories:

```
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

Install the worker binaries:

```
{
  mkdir containerd
  tar -xvf crictl-v1.15.0-linux-amd64.tar.gz
  tar -xvf containerd-1.2.9.linux-amd64.tar.gz -C containerd
  sudo tar -xvf cni-plugins-linux-amd64-v0.8.2.tgz -C /opt/cni/bin/
  sudo mv runc.amd64 runc
  chmod +x crictl kubectl kubelet runc
  sudo mv crictl kubectl kubelet runc /usr/local/bin/
  sudo mv containerd/bin/* /bin/
}
```

### Configure CNI Networking

This will be done automatically when deploying `kube-router`.

### Configure containerd

Create the `containerd` configuration file:

```
sudo mkdir -p /etc/containerd/
```

```
cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
EOF
```

Create the `containerd.service` systemd unit file:

```
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubelet

```
{
  SHORTNAME=$(hostname -s)
  sudo mv ${SHORTNAME}-key.pem ${SHORTNAME}.pem /var/lib/kubelet/
  sudo mv ${SHORTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
  sudo mv ca.pem /var/lib/kubernetes/
}
```

Create the `kubelet-config.yaml` configuration file:

```
SHORTNAME=$(hostname -s)

cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
resolvConf: "/etc/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${SHORTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${SHORTNAME}-key.pem"
EOF
```

> The `resolvConf` configuration is used to avoid loops when using CoreDNS for service discovery on systems running `systemd-resolved`.

Create the `kubelet.service` systemd unit file:

```
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Start the Worker Services

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable containerd kubelet --now
}
```

> Remember to run the above commands on each worker node: `worker-0`, `worker-1`, and `worker-2`.

## Verification

> The compute instances created in this tutorial will not have permission to complete this section. Run the following commands from the same machine used to create the compute instances.

List the registered Kubernetes nodes:

```
ssh -i ~/.ssh/k8s.pem centos@controller-0.${DOMAIN} \
  "kubectl get nodes --kubeconfig admin.kubeconfig"
```

> output

```
NAME               STATUS     ROLES    AGE   VERSION
worker-0.k8s.lan   NotReady   <none>   17s   v1.15.3
worker-1.k8s.lan   NotReady   <none>   17s   v1.15.3
worker-2.k8s.lan   NotReady   <none>   17s   v1.15.3
```

**NOTE:** The nodes are 'NotReady' because there is no CNI configured yet.
This will be fixed in the "Pod Network Routes" chapter.

Next: [Configuring kubectl for Remote Access](10-configuring-kubectl.md)
