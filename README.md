# etcd operator
unit/integration:
[![Build Status](https://jenkins-etcd-public.prod.coreos.systems/view/operator/job/etcd-operator-unit-master/badge/icon)](https://jenkins-etcd-public.prod.coreos.systems/view/operator/job/etcd-operator-unit-master/)
e2e (Kubernetes stable):
[![Build Status](https://jenkins-etcd-public.prod.coreos.systems/buildStatus/icon?job=etcd-operator-master)](https://jenkins-etcd-public.prod.coreos.systems/job/etcd-operator-master/)
e2e (Kubernetes master):
[![Build Status](https://jenkins-etcd-public.prod.coreos.systems/buildStatus/icon?job=etcd-operator-master-k8s-master)](https://jenkins-etcd-public.prod.coreos.systems/job/etcd-operator-master-k8s-master/)

### Project status: beta

Major planned features have been completed and while no breaking API changes are currently planned, we reserve the right to address bugs and API changes in a backwards incompatible way before the project is declared stable. Any breaking changes introduced will be documented in release notes.

Currently user face etcd cluster objects are created as [Kubernetes Third Party Resources](https://kubernetes.io/docs/user-guide/thirdpartyresources/), however, taking advantage of [User Aggregated API Servers](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/aggregated-api-servers.md) to improve reliability, validation and versioning is planned. The use of Aggregated API should be minimally disruptive to existing users but may change what Kubernetes objects are created or how users deploy the etcd operator.

We expect to consider the etcd operator stable soon; backwards incompatible changes will not be made once the project reaches stability.

## Overview

The etcd operator manages etcd clusters deployed to [Kubernetes][k8s-home] and automates tasks related to operating an etcd cluster.

- [Create and destroy](#create-and-destroy-an-etcd-cluster)
- [Resize](#resize-an-etcd-cluster)
- [Recover a member](#member-recovery)
- [Backup and restore a cluster](#disaster-recovery)
- [Rolling upgrade](#upgrade-an-etcd-cluster)

Read [Best Practices](./doc/best_practices.md) for more information on how to better use etcd operator.

Read [RBAC docs](./doc/user/rbac.md) for how to setup RBAC rules for etcd operator if RBAC is in place.

Read [Developer Guide](./doc/dev/developer_guide.md) for setting up development environment if you want to contribute.

## Requirements

- Kubernetes 1.5.3+
- etcd 3.0+

## Demo

![etcd Operator demo](https://raw.githubusercontent.com/coreos/etcd-operator/master/doc/gif/demo.gif)

## Deploy etcd operator

See [instructions on how to install/uninstall etcd operator](doc/user/op_guide.md) .

## Create and destroy an etcd cluster

```bash
$ kubectl create -f example/example-etcd-cluster.yaml
```

A 3 member etcd cluster will be created.

```bash
$ kubectl get pods
NAME                            READY     STATUS    RESTARTS   AGE
example-etcd-cluster-0000       1/1       Running   0          1m
example-etcd-cluster-0001       1/1       Running   0          1m
example-etcd-cluster-0002       1/1       Running   0          1m

$ kubectl get services
NAME                        CLUSTER-IP    EXTERNAL-IP   PORT(S)             AGE
example-etcd-cluster-0000   10.0.6.23     <none>        2380/TCP,2379/TCP   2m
example-etcd-cluster-0001   10.0.64.204   <none>        2380/TCP,2379/TCP   1m
example-etcd-cluster-0002   10.0.199.80   <none>        2380/TCP,2379/TCP   1m
```

If you are working with [minikube locally](https://github.com/kubernetes/minikube#minikube) create a nodePort service and test out that etcd is responding:

```bash
$ kubectl create -f example/example-etcd-cluster-nodeport-service.json
$ export ETCDCTL_API=3
$ export ETCDCTL_ENDPOINTS=$(minikube service example-etcd-cluster-client-service --url)
$ etcdctl put foo bar
```

Destroy etcd cluster:

```bash
$ kubectl delete -f example/example-etcd-cluster.yaml
```

## Resize an etcd cluster

`kubectl apply` doesn't work for TPR at the moment. See [kubernetes/#29542](https://github.com/kubernetes/kubernetes/issues/29542).
As a workaround, we use cURL to resize the cluster.

Create an etcd cluster:

```
$ kubectl create -f example/example-etcd-cluster.yaml
```

Use kubectl to create a reverse proxy:

```
$ kubectl proxy --port=8080
Starting to serve on 127.0.0.1:8080
```
Now we can talk to apiserver via "http://127.0.0.1:8080".

Create a json file with the new configuration:

```
$ cat body.json
{
  "apiVersion": "etcd.coreos.com/v1beta1",
  "kind": "Cluster",
  "metadata": {
    "name": "example-etcd-cluster",
    "namespace": "default"
  },
  "spec": {
    "size": 5
  }
}
```

In another terminal, use the following command to change the cluster size from 3 to 5.

```
$ curl -H 'Content-Type: application/json' -X PUT --data @body.json http://127.0.0.1:8080/apis/etcd.coreos.com/v1beta1/namespaces/default/clusters/example-etcd-cluster
```

We should see

```
$ kubectl get pods
NAME                            READY     STATUS    RESTARTS   AGE
example-etcd-cluster-0000       1/1       Running   0          1m
example-etcd-cluster-0001       1/1       Running   0          1m
example-etcd-cluster-0002       1/1       Running   0          1m
example-etcd-cluster-0003       1/1       Running   0          1m
example-etcd-cluster-0004       1/1       Running   0          1m
```

Now we can decrease the size of cluster from 5 back to 3.

Create a json file with cluster size of 3:

```
$ cat body.json
{
  "apiVersion": "etcd.coreos.com/v1beta1",
  "kind": "Cluster",
  "metadata": {
    "name": "example-etcd-cluster",
    "namespace": "default"
  },
  "spec": {
    "size": 3
  }
}
```

Apply it to API Server:

```
$ curl -H 'Content-Type: application/json' -X PUT --data @body.json http://127.0.0.1:8080/apis/etcd.coreos.com/v1beta1/namespaces/default/clusters/example-etcd-cluster
```

We should see that etcd cluster will eventually reduce to 3 pods:

```
$ kubectl get pods
NAME                            READY     STATUS    RESTARTS   AGE
example-etcd-cluster-0002       1/1       Running   0          1m
example-etcd-cluster-0003       1/1       Running   0          1m
example-etcd-cluster-0004       1/1       Running   0          1m
```

## Member recovery

If the minority of etcd members crash, the etcd operator will automatically recover the failure.
Let's walk through in the following steps.

Create an etcd cluster:

```
$ kubectl create -f example/example-etcd-cluster.yaml
```

Wait until all three members are up. Simulate a member failure by deleting a pod:

```bash
$ kubectl delete pod example-etcd-cluster-0000 --now
```

The etcd operator will recover the failure by creating a new pod `example-etcd-cluster-0003`:

```bash
$ kubectl get pods
NAME                            READY     STATUS    RESTARTS   AGE
example-etcd-cluster-0001       1/1       Running   0          1m
example-etcd-cluster-0002       1/1       Running   0          1m
example-etcd-cluster-0003       1/1       Running   0          1m
```

Destroy etcd cluster:
```bash
$ kubectl delete -f example/example-etcd-cluster.yaml
```

## etcd operator recovery

If the etcd operator restarts, it can recover its previous state.
Let's walk through in the following steps.

```
$ kubectl create -f example/example-etcd-cluster.yaml
```

Wait until all three members are up. Then

```bash
$ kubectl delete -f example/deployment.yaml
deployment "etcd-operator" deleted

$ kubectl delete pod example-etcd-cluster-0000 --now
pod "example-etcd-cluster-0000" deleted
```

Then restart the etcd operator. It should recover itself and the etcd clusters it manages.

```bash
$ kubectl create -f example/deployment.yaml
deployment "etcd-operator" created

$ kubectl get pods
NAME                            READY     STATUS    RESTARTS   AGE
example-etcd-cluster-0001       1/1       Running   0          1m
example-etcd-cluster-0002       1/1       Running   0          1m
example-etcd-cluster-0003       1/1       Running   0          1m
```

## Disaster recovery

If the majority of etcd members crash, but at least one backup exists for the cluster, the etcd operator can restore the entire cluster from the backup.

By default, the etcd operator creates a storage class on initialization:

```
$ kubectl get storageclass
NAME                 TYPE
etcd-backup-gce-pd   kubernetes.io/gce-pd
```

This is used to request the persistent volume to store the backup data. See [other backup options](doc/user/backup_options.md).

To enable backup, create an etcd cluster with [backup enabled spec](example/example-etcd-cluster-with-backup.yaml).

```
$ kubectl create -f example/example-etcd-cluster-with-backup.yaml
```

A persistent volume claim is created for the backup pod:

```
$ kubectl get pvc
NAME                                   STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
example-etcd-cluster-with-backup-pvc   Bound     pvc-79e39bab-b973-11e6-8ae4-42010af00002   1Gi        RWO           9s
```

Let's try to write some data into etcd:

```
$ kubectl run --rm -i --tty fun --image quay.io/coreos/etcd --restart=Never -- /bin/sh
/ # ETCDCTL_API=3 etcdctl --endpoints http://example-etcd-cluster-with-backup-0002:2379 put foo bar
OK
(ctrl-D to exit)
```

Now let's kill two pods to simulate a disaster failure:

```
$ kubectl delete pod example-etcd-cluster-with-backup-0000 example-etcd-cluster-with-backup-0001 --now
pod "example-etcd-cluster-with-backup-0000" deleted
pod "example-etcd-cluster-with-backup-0001" deleted
```

Now quorum is lost. The etcd operator will start to recover the cluster by:
- Creating a new seed member to recover from the backup
- Add more members until the size reaches to specified number

```
$ kubectl get pods
NAME                                                    READY     STATUS     RESTARTS   AGE
example-etcd-cluster-with-backup-0003                   0/1       Init:0/2   0          11s
example-etcd-cluster-with-backup-backup-sidecar-e9gkv   1/1       Running    0          18m
...
$ kubectl get pods
NAME                                                    READY     STATUS    RESTARTS   AGE
example-etcd-cluster-with-backup-0003                   1/1       Running   0          3m
example-etcd-cluster-with-backup-0004                   1/1       Running   0          3m
example-etcd-cluster-with-backup-0005                   1/1       Running   0          3m
example-etcd-cluster-with-backup-backup-sidecar-e9gkv   1/1       Running   0          22m
```

Finally, besides destroying the cluster, also cleanup the backup if you don't need it anymore:
```
$ kubectl delete pvc example-etcd-cluster-with-backup-pvc
```

Note: There could be a race that it will fall to single member recovery if a pod is recovered before another is deleted.

## Upgrade an etcd cluster

Have the following yaml file ready:

```
$ cat 3.0-etcd-cluster.yaml
apiVersion: "etcd.coreos.com/v1beta1"
kind: "Cluster"
metadata:
  name: "example-etcd-cluster"
spec:
  size: 3
  version: "3.0.16"
```

Create an etcd cluster with the version specified (3.0.16) in the yaml file:

```
$ kubectl create -f 3.0-etcd-cluster.yaml
$ kubectl get pods
NAME                           READY     STATUS    RESTARTS   AGE
example-etcd-cluster-0000      1/1       Running   0          37s
example-etcd-cluster-0001      1/1       Running   0          25s
example-etcd-cluster-0002      1/1       Running   0          14s
```

The container image version should be 3.0.16:

```
$ kubectl get pod example-etcd-cluster-0000 -o yaml | grep "image:" | uniq
    image: quay.io/coreos/etcd:v3.0.16
```

`kubectl apply` doesn't work for TPR at the moment. See [kubernetes/#29542](https://github.com/kubernetes/kubernetes/issues/29542).
We use cURL to update the cluster as a workaround.

Use kubectl to create a reverse proxy:

```
$ kubectl proxy --port=8080
Starting to serve on 127.0.0.1:8080
```

Have following json file ready:
(Note that the version field is changed from 3.0.16 to 3.1.2)

```
$ cat body.json
{
  "apiVersion": "etcd.coreos.com/v1beta1",
  "kind": "Cluster",
  "metadata": {
    "name": "example-etcd-cluster"
  },
  "spec": {
    "size": 3,
    "version": "3.1.2"
  }
}
```

Then we update the version in spec.

```
$ curl -H 'Content-Type: application/json' -X PUT --data @body.json \
    http://127.0.0.1:8080/apis/etcd.coreos.com/v1beta1/namespaces/default/clusters/example-etcd-cluster
```

Wait ~30 seconds. The container image version should be updated to v3.1.2:

```
$ kubectl get pod example-etcd-cluster-0000 -o yaml | grep "image:" | uniq
    image: quay.io/coreos/etcd:v3.1.2
```

Check the other two pods and you should see the same result.

## Limitations

- The etcd operator only manages the etcd cluster created in the same namespace. Users need to create multiple operators in different namespaces to manage etcd clusters in different namespaces.

- Backup works only for data in etcd3 storage, not for data in etcd2 storage.

- Backup requires PV to work, and it only works on GCE(kubernetes.io/gce-pd) and AWS(kubernetes.io/aws-ebs) for now.

- Migration, the process of allowing the etcd operator to manage existing etcd3 clusters, only supports a single-member cluster, with all nodes running in the same Kubernetes cluster.

**The operator collects anonymous usage statistics to help us learn how the software is being used and how we can improve it. To disable collection, run the operator with the flag `-analytics=false`.**


[k8s-home]: http://kubernetes.io
