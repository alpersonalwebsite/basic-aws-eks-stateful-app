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

Measured: a 5.6 datadir, then MySQL 8.0 pointed at it, container exit 1.

MySQL's supported path is 5.6 → 5.7 → 8.0, one major at a time. For a walkthrough whose
data is disposable, do not do that. Delete the claim and start clean:

```shell
kubectl delete deployment wordpress wordpress-mysql
kubectl delete pvc mysql-pv-claim wp-pv-claim
```

Then re-apply. A fresh volume is initialised by MySQL 8.0 and the problem does not arise.
If the data is *not* disposable, migrate through 5.7 before changing the image.

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

