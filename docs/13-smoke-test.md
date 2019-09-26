# Smoke Test

In this lab you will complete a series of tasks to ensure your Kubernetes cluster is functioning correctly.

## Data Encryption

In this section you will verify the ability to [encrypt secret data at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#verifying-that-data-is-encrypted).

Create a generic secret:

```
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

Print a hexdump of the `kubernetes-the-hard-way` secret stored in etcd:

```
ssh -i ~/.ssh/k8s.pem centos@controller-0.${DOMAIN} \
  "sudo ETCDCTL_API=3 /usr/local/bin/etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem\
  /registry/secrets/default/kubernetes-the-hard-way | hexdump -C"
```

> output

```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a 43 99 9c e0 a3 e2 15  |:v1:key1:C......|
00000050  57 f9 e7 2e f8 a1 69 84  93 3d 4d b4 83 44 a1 03  |W.....i..=M..D..|
00000060  1b 46 3a e9 e6 c0 40 1d  67 0a 30 5e 26 85 c8 7d  |.F:...@.g.0^&..}|
00000070  b3 8b 0d ce 5b 33 87 4e  af 91 37 cc eb 06 7c bb  |....[3.N..7...|.|
00000080  09 74 34 08 db 71 d4 fb  25 f8 0d ca 9f 72 48 c2  |.t4..q..%....rH.|
00000090  14 b2 40 a9 f3 b9 35 56  29 7a 99 4a de f1 26 1c  |..@...5V)z.J..&.|
000000a0  2e 5f 5b 16 30 e9 48 53  78 ee 82 52 a0 e9 b7 4f  |._[.0.HSx..R...O|
000000b0  a7 83 4c b5 ef 51 b4 f2  52 49 fc c6 e0 40 30 c1  |..L..Q..RI...@0.|
000000c0  11 44 20 78 46 12 f9 eb  77 4e 05 5a c7 ab d3 ae  |.D xF...wN.Z....|
000000d0  da d7 92 8c a4 ba 10 df  ed b6 fa cf 4b ec db 59  |............K..Y|
000000e0  b1 92 6c 14 a8 c1 1a da  ed 0a                    |..l.......|
000000ea
```

The etcd key should be prefixed with `k8s:enc:aescbc:v1:key1`, which indicates the `aescbc` provider was used to encrypt the data with the `key1` encryption key.

## Deployments

In this section you will verify the ability to create and manage [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Create a deployment for the [nginx](https://nginx.org/en/) web server:

```
kubectl create deployment nginx --image=nginx
```

List the pod created by the `nginx` deployment:

```
kubectl get pods -l run=nginx
```

> output

```
NAME                     READY     STATUS    RESTARTS   AGE
nginx-65899c769f-xkfcn   1/1       Running   0          15s
```

### Port Forwarding

In this section you will verify the ability to access applications remotely using [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/).

Retrieve the full name of the `nginx` pod:

```
POD_NAME=$(kubectl get pods -l run=nginx -o jsonpath="{.items[0].metadata.name}")
```

Forward port `8080` on your local machine to port `80` of the `nginx` pod:

```
kubectl port-forward $POD_NAME 8080:80
```

> output

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

In a new terminal make an HTTP request using the forwarding address:

```
curl --head http://127.0.0.1:8080
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.17.4
Date: Thu, 26 Sep 2019 12:27:10 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 24 Sep 2019 14:49:10 GMT
Connection: keep-alive
ETag: "5d8a2ce6-264"
Accept-Ranges: bytes
```

Switch back to the previous terminal and stop the port forwarding to the `nginx` pod:

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
^C
```

### Logs

In this section you will verify the ability to [retrieve container logs](https://kubernetes.io/docs/concepts/cluster-administration/logging/).

Print the `nginx` pod logs:

```
kubectl logs $POD_NAME
```

> output

```
127.0.0.1 - - [26/Sep/2019:12:27:10 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.65.3" "-"
```

### Exec

In this section you will verify the ability to [execute commands in a container](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#running-individual-commands-in-a-container).

Print the nginx version by executing the `nginx -v` command in the `nginx` container:

```
kubectl exec -ti $POD_NAME -- nginx -v
```

> output

```
nginx version: nginx/1.17.4
```

## Services

In this section you will verify the ability to expose applications using a [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

Expose the `nginx` deployment using a [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) service:

```
kubectl expose deployment nginx --port 80 --type NodePort
```

> The LoadBalancer service type can not be used because your cluster is not configured with [cloud provider integration](https://kubernetes.io/docs/getting-started-guides/scratch/#cloud-provider). Setting up cloud provider integration is out of scope for this tutorial.

Retrieve the node port assigned to the `nginx` service:

```
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

Create a security group rule that allows remote access to the `nginx` node port:

```
openstack security group rule create \
  --ingress \
  --protocol tcp \
  --dst-port ${NODE_PORT} \
  kubernetes-the-hard-way-allow-external
```

Retrieve the external IP address of a worker instance:

```
EXTERNAL_IP=$(openstack server show worker-0.${DOMAIN} -f value -c addresses | awk '{ print $2 }')
```

Make an HTTP request using the external IP address and the `nginx` node port:

```
curl -I http://${EXTERNAL_IP}:${NODE_PORT}
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.17.4
Date: Thu, 26 Sep 2019 12:39:19 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 24 Sep 2019 14:49:10 GMT
Connection: keep-alive
ETag: "5d8a2ce6-264"
Accept-Ranges: bytes
```

Next: [Cleaning Up](14-cleanup.md)
