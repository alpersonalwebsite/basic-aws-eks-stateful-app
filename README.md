# Deploying a Stateful App

<!-- TODO:
  Differences between EBS and EFS
-->

Note: 
* EBS is tied to a particular AZ.
* EFS can be accessed from different AZ through a Security Group.

## The sample app was updated, and it needed more than a tag bump

The manifests deployed `wordpress:4.8-apache` and `mysql:5.6`. Pulled and inspected, that
WordPress image is **WordPress 4.8.3 running PHP 5.6.32**, built November 2017; PHP 5.6
has been end of life since December 2018, and MySQL 5.6 since February 2021. Both tags
still pull, so this kept working, but it is not something to stand up on a cluster.

They are `wordpress:6-apache` and `mysql:8.0` now. Swapping the tags **on its own returns
a 500**, and finding out why took running it:

| what broke | why |
| --- | --- |
| `Error establishing a database connection`, no `wordpress` schema | WordPress 6's entrypoint does not create the database the way 4.8's did. MySQL has to, via `MYSQL_DATABASE`. |
| `Error establishing a database connection`, database present | The 4.8 image defaulted `DB_USER` to `root`. The current image defaults it to the literal string `'example username'`, so `WORDPRESS_DB_USER` must be set. |

Of the two WordPress variables, only `WORDPRESS_DB_USER` is actually required:
`WORDPRESS_DB_NAME` already defaults to `wordpress` in this image, and omitting it still
serves the installer (checked). It is set anyway so it pairs visibly with
`MYSQL_DATABASE` and a future change to that default cannot quietly break the walkthrough.

So the manifests also gained `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD`,
`WORDPRESS_DB_NAME` and `WORDPRESS_DB_USER`. A side benefit: WordPress now connects as a
dedicated `wordpress` user instead of as MySQL `root`, on its **own** password. Sharing
one secret key between root and the application would have meant that anything leaking the
application's password had leaked root's too, so the secret now carries two keys and the
`kubectl create secret` examples generate both.

### If you already ran this with MySQL 5.6, delete the volume first

MySQL 8.0 cannot open a 5.6 data directory. Applying the new manifest over an existing
`mysql-pv-claim` leaves the pod in CrashLoopBackOff, and the reason is only in the logs:

```text
[ERROR] [MY-013090] [InnoDB] Unsupported redo log format (v0). The redo log was created before MySQL 5.7.9
[ERROR] [MY-011013] [Server] Failed to initialize DD Storage Engine.
[ERROR] [MY-010020] [Server] Data Dictionary initialization failed.
```

Measured: a 5.6 datadir, then MySQL 8.0 pointed at it, the container exited with code 1.

MySQL's supported path is 5.6 → 5.7 → 8.0, one major at a time. For a walkthrough whose
data is disposable, do not do that. Delete the claim and start clean:

The walkthroughs run in their own namespaces, so the commands need `-n`. Without it these
run against whatever your current context is, which is not where the resources are. They do
not fail silently, and the error is the useful part:

```text
Error from server (NotFound): deployments.apps "wordpress" not found
Error from server (NotFound): deployments.apps "wordpress-mysql" not found
```

Exit status 1. Measured on Kubernetes 1.30 — but do not read too much into it. `NotFound` is
ambiguous three ways: the namespace does not exist, it exists and is empty, or you are in the
right namespace and the deployments really were already deleted. The message is identical in all
three, so it cannot tell you which you are in. Check before concluding:

```shell
kubectl get namespace dev-stateful-ebs

kubectl get deployment,pvc,secret -n dev-stateful-ebs
```

If the namespace is missing, you are in the wrong place or it is already gone. If it exists and
lists nothing, the cleanup is done. An earlier version of this README claimed `NotFound` meant
"you are looking in the wrong place"; that is one of the three cases, not the meaning.

**EBS variant:**

```shell
kubectl -n dev-stateful-ebs delete deployment wordpress wordpress-mysql
kubectl -n dev-stateful-ebs delete pvc mysql-pv-claim wp-pv-claim
```

**EFS variant** — and deleting the claim is **not enough here**:

```shell
kubectl -n dev-stateful-efs delete deployment wordpress wordpress-mysql
kubectl -n dev-stateful-efs delete pvc efs-pv-claim
```

`efs/eks/pv.yaml` sets `persistentVolumeReclaimPolicy: Retain` against a fixed
`volumeHandle`, so the data survives the claim. Two consequences, and the first one is a step
this README used to omit.

**The retained PV will not accept a new claim.** Under `Retain`, deleting the claim moves the PV
to `Released` and it *keeps* `spec.claimRef`, pinned to the deleted claim's UID. A replacement
PVC has a new UID, so it never binds and sits in `Pending` indefinitely. Measured on Kubernetes
1.31:

```text
after deleting the claim:  PV Released, claimRef efs-pv-claim uid 24e8cd65…
new PVC applied:           uid 42c8beaf…  (different)
new PVC after 90s:         Pending
after removing only spec.claimRef from the PV:  PV Bound, PVC Bound
```

That last line is the control: nothing else changed, so the stale `claimRef` is the cause. So
release the PV before re-applying, either by deleting and recreating it with the same
`volumeHandle`:

```shell
kubectl delete pv efs-psv-eks
kubectl apply -f efs/eks/pv.yaml
```

or by clearing the reference on the existing one:

```shell
kubectl patch pv efs-psv-eks --type=json -p '[{"op":"remove","path":"/spec/claimRef"}]'
```

**Releasing the PV does not touch the data.** The file system and its contents survive, so once
the claim binds again MySQL 8.0 finds the same 5.6 directory and fails exactly as before. When
the data is disposable, empty the MySQL directory on the file system itself — mount it from a
helper pod or an EC2 instance in the same VPC and remove its contents — before re-applying. Note
that with the `subPath` change the MySQL files live under a `mysql/` directory on the volume
rather than at its root.

Then re-apply. A freshly initialised volume is created by MySQL 8.0 and the problem does not
arise. If the data is *not* disposable, migrate through 5.7 before changing the image.

**The EFS variant shared one directory between MySQL and Apache.** Both deployments mount the
same `efs-pv-claim`, which is the point of EFS being ReadWriteMany, but neither mount set
`subPath`, so both landed on the volume's *root*: `/var/lib/mysql` and `/var/www/html` were the
same directory and the database files sat inside the web server's document root. Measured on
`wordpress:6.7-apache`, a file written into `/var/www/html` is served with **HTTP 200 and its
contents**, so `GET /ibdata1` returns the InnoDB tablespace and `GET /wordpress/wp_users.ibd` the
users table, with no authentication. Both mounts now carry a `subPath` (`mysql` and `wordpress`),
which keeps them in separate directories on the one file system. The EBS variant was never
affected, since it uses two distinct claims.

That changes the on-volume layout: an existing file system has the data at its root, and after
this change the pods look under `mysql/` and `wordpress/`. For a disposable walkthrough, start
from an empty file system.

Verified end-to-end on a real Kubernetes 1.30 cluster with these exact files: both pods
Running, `GET /` returns **200** and serves the WordPress installer. `kubeconform` reports
13 of 13 resources valid across both variants.

Two notes on what that verification does **not** cover. It ran on k3s with its local-path
storage, not on EKS with EBS or EFS, so the storage-class and volume plumbing in these
walkthroughs is unchanged and untested here. And the passwords in the `kubectl create
secret` examples now generate their values with `openssl rand` rather than committing
literal passwords to a public repository.

One upside not obvious from the diff: `mysql:5.6` was published for `linux/amd64` only,
while `mysql:8.0` ships `arm64` as well, so these manifests now also run on Graviton node
groups.

## Pre reqs

Please, be sure you followed the steps in [Basic AWS EKS](https://github.com/alpersonalwebsite/basic-aws-eks) and you have your cluster up and running.

### TL;DR

**Create cluster and nodegorup**
```shell
eksctl create cluster -f basic-aws-eks/eks/cluster-autoscaling.yaml 
```

This is going to create both stacks:
* eksctl-basic-eks-cluster-cluster
* eksctl-basic-eks-cluster-nodegroup-ng-2

Then, follow the instructions for...
* Create deployment for AutoScaler
* HELM Package Manager
* EKS and users

[EBS example](./ebs/stateful-app-with-ebs.md)

[EFS example](./efs/stateful-app-with-efs.md)

