# Deploying a Stateful App with EBS

We are deploying WordPress with MySQL Db using EBS to store data (of both)

This example consists of the following components:

* WP as frontend (with EBS to store data). WP pod will connect to MySQL pod thorugh kubernetes MySQL Service.
* MySQL as backend (with EBS to store data)
* A Load Balancer which is going to expose our app

## Create namespace

```shell
kubectl create namespace dev-stateful-ebs
```

Output:

```shell
namespace/dev-stateful-ebs created
```

## Create persistent volume
We are going to ujse the default storage class (gp2)

```shell
kubectl apply -f eks/pvcs.yaml --namespace=dev-stateful-ebs
```

Output:

```shell
persistentvolumeclaim/mysql-pv-claim created
persistentvolumeclaim/wp-pv-claim created
```

## Backend

### Create secret for MySQL password

```shell
kubectl create secret generic mysql-password --from-literal=password=$(openssl rand -base64 24) \
  --from-literal=wordpress-password=$(openssl rand -base64 24) --namespace=dev-stateful-ebs 
```

Output:

```shell
secret/mysql-password created
```

We can check the secrets for our namespace:

```shell
kubectl get secrets --namespace=dev-stateful-ebs
```

Output:

```shell
NAME                  TYPE                                  DATA   AGE
default-token-k7cmf   kubernetes.io/service-account-token   3      14m
mysql-password        Opaque                                2      85s
```

### Create Service and Deployment for MySQL

```shell
kubectl apply -f eks/service-and-deployment-mysql.yaml --namespace=dev-stateful-ebs 
```

Output:

```shell
service/wordpress-mysql created
deployment.apps/wordpress-mysql created
```

## Frontend

<!-- 
TODO:
Difference between Deployment and StatefulSet
-->

### Deploy via Deployment
Ww are going to have multiples pods (inside a kubernetes node) accessing one EBS volume.

#### Create Service and Deployment for WP

```shell
kubectl apply -f eks/service-and-deployment-wp.yaml --namespace=dev-stateful-ebs
```

Output:

```shell
service/wordpress created
deployment.apps/wordpress created
```

#### List persistent volumes for namespace

```shell
kubectl get pvc --namespace=dev-stateful-ebs
```

Sample ouput:

```shell
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-pv-claim   Bound    pvc-88172a08-22f9-4523-8cf0-701ef88dd04d   20Gi       RWO            gp2            6d22h
wp-pv-claim      Bound    pvc-75f605cd-9b97-4315-ae46-0bfa13d4f5c2   20Gi       RWO            gp2            6d22h
```

Then, we can go to `Load Balancers` in `AWS Console`: https://us-west-1.console.aws.amazon.com/ec2/v2/home?region=us-west-1#LoadBalancers:sort=desc:createdTime
and use the DNS name (example: ab1c59458a8c0419e83fdbd3516bc0b4-1774800410.us-west-1.elb.amazonaws.com) to set up our `WP site`

<!--
### Deploy via StatefulSet
For each pod we are going to have its own persistent volume.

NOT IDEAL FOR WORDPRESS
-->

## Clean up

### Frontend

```shell
kubectl delete -f eks/service-and-deployment-wp.yaml --namespace=dev-stateful-ebs
```

Output:

```shell
service "wordpress" deleted
deployment.apps "wordpress" deleted
```

### Backend

```shell
kubectl delete -f eks/service-and-deployment-mysql.yaml --namespace=dev-stateful-ebs 
```

Output:

```shell
service "wordpress-mysql" deleted
deployment.apps "wordpress-mysql" deleted
```

### EBS volumes

**Delete the claim, and the rest follows. Do not start in the console.** The sample output
below shows these PVs with `RECLAIM POLICY: Delete`, which is what the default `gp2`
StorageClass sets. Under that policy, deleting the PVC releases the PV, Kubernetes deletes the
PV, and the EBS CSI driver deletes the backing volume. One command covers all three:

```shell
kubectl -n dev-stateful-ebs delete pvc mysql-pv-claim wp-pv-claim
```

The `-n` is not optional: both claims live in `dev-stateful-ebs`, so without it this runs
against your current context and reports `NotFound` while the claims survive.

Then confirm, rather than assuming:

```shell
kubectl get pv

aws ec2 describe-volumes --region us-west-1 \
  --filters "Name=tag:kubernetes.io/created-for/pvc/namespace,Values=dev-stateful-ebs" \
  --query 'Volumes[].{id:VolumeId,state:State}' --output table
```

<details>
<summary>What this section used to say, and why it left PVs stuck in Terminating</summary>

The original deleted the EBS volumes in the EC2 console first, then ran `kubectl delete pv`
directly on PVs whose PVCs still existed. A PV cannot finish deleting while a claim is bound
to it, so both sat in `Terminating`:

```text
kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS        CLAIM                                    STORAGECLASS   REASON   AGE
pvc-28f1129c-ad1d-45df-8e52-8ab24c578ce5   8Gi        RWO            Delete           Bound         default/redis-data-redis-test-slave-0    gp2                     13d
pvc-4018b8ff-ff94-4ed9-8c5f-3148f77b7169   8Gi        RWO            Delete           Bound         default/redis-data-redis-test-master-0   gp2                     13d
pvc-75f605cd-9b97-4315-ae46-0bfa13d4f5c2   20Gi       RWO            Delete           Terminating   dev-stateful-ebs/wp-pv-claim             gp2                     6d22h
pvc-7d93df27-9460-4fb5-9790-884aba63237b   8Gi        RWO            Delete           Bound         default/redis-data-redis-test-slave-1    gp2                     13d
pvc-88172a08-22f9-4523-8cf0-701ef88dd04d   20Gi       RWO            Delete           Terminating   dev-stateful-ebs/mysql-pv-claim          gp2                     6d22h
```

It then cleared the finalizer by hand, described as setting the status to lost:

```text
kubectl patch pv pvc-88172a08-22f9-4523-8cf0-701ef88dd04d -p '{"metadata":{"finalizers":null}}'
kubectl patch pv pvc-75f605cd-9b97-4315-ae46-0bfa13d4f5c2 -p '{"metadata":{"finalizers":null}}'
```

That deletes the API object without releasing anything, and it is only ever needed because the
deletion happened in the wrong order. It also hides a real failure: if the CSI driver could not
delete the volume, clearing the finalizer removes the evidence and you keep paying for the EBS
volume. Both blocks are quoted here so the old instructions are recognisable, not to be run.

The PV names above are from the 2022 run and are specific to that cluster, which is the other
reason to delete claims by name instead: `mysql-pv-claim` and `wp-pv-claim` are stable, and
`pvc-88172a08-...` is not.

</details>

**Not re-verified against EBS.** The corrected order is the documented behaviour of the
`Delete` reclaim policy, and the repository's verification ran on k3s with local-path storage,
not on EKS with EBS. See the README for what was and was not exercised.

### Delete secret

```shell
kubectl delete secret mysql-password --namespace=dev-stateful-ebs
```

The secret was created with `--namespace=dev-stateful-ebs`, so deleting it needs the same flag.
Without it, measured on Kubernetes 1.31: `Error from server (NotFound): secrets
"mysql-password" not found`, exit status 1, and the secret holding the MySQL root password and
the WordPress database password stays in the cluster.

**The following part is the DELETE section of `basic-aws-eks`**

### Delete nodegroup

This would be the same to `delete in CFN` the stack `eksctl-basic-eks-cluster-nodegroup-ng-2`

```shell
eksctl delete nodegroup --config-file=eks/cluster-autoscaling.yaml --approve 
```

Output:

```
...
...
...
2022-08-09 09:57:33 [✔]  deleted 1 nodegroup(s) from cluster "basic-eks-cluster"
```

### Delete cluster

This would be the same to `delete in CFN` the stack `eksctl-basic-eks-cluster-cluster`

Before doing this, be sure that there are no resources tied to the VPC: example, Security Groups.

```shell
eksctl delete cluster -f eks/cluster-autoscaling.yaml
```

Output:

```text
2022-08-09 10:02:07 [ℹ]  deleting EKS cluster "basic-eks-cluster"
2022-08-09 10:02:07 [ℹ]  deleted 0 Fargate profile(s)
2022-08-09 10:02:08 [✔]  kubeconfig has been updated
2022-08-09 10:02:08 [ℹ]  cleaning up AWS load balancers created by Kubernetes objects of Kind Service or Ingress
2022-08-09 10:02:09 [ℹ]  1 task: { delete cluster control plane "basic-eks-cluster" [async] }
2022-08-09 10:02:09 [ℹ]  will delete stack "eksctl-basic-eks-cluster-cluster"
2022-08-09 10:02:09 [✔]  all cluster resources were deleted
```

### Delete the CFN stack

This is the step that removes the IAM users and their access keys. It used to name
`service-support`, a stack this project never creates, so the documented cleanup left the users
and any keys made for them active.

**Delete the access keys first, then the stack.** The order is not cosmetic. A stack delete
tears down the `AWS::IAM::User` resources, and per AWS's `DeleteUser` API reference, "when you
delete a user programmatically, you must delete the items attached to the user manually, or the
deletion fails", with access keys named explicitly and `DeleteConflict` (HTTP 409) as the error.
Keys you created by hand are not part of the stack, so leaving them in place can fail the
delete and leave the users, and their keys, in the account.

The usernames come off the deployed stack rather than being hard-coded, because
`eks-operator`, `eks-admin-user` and `eks-user` are only the parameter *defaults* in
`cfn/eks-project.yml`. That is a second reason this runs before the delete: the stack must still
exist to be queried.

```shell
aws cloudformation describe-stacks \
  --stack-name eks-project \
  --region us-west-1 \
  --query "Stacks[0].Parameters[?ParameterKey=='EKSUserName'||ParameterKey=='EKSAdminUserName'||ParameterKey=='EKSRegularUserName'].ParameterValue" \
  --output text \
  | tr '\t' '\n' \
  | while read -r u; do
      [ -n "$u" ] || continue
      for k in $(aws iam list-access-keys --user-name "$u" \
                   --query 'AccessKeyMetadata[].AccessKeyId' --output text 2>/dev/null); do
        echo "deleting access key $k for $u"
        aws iam delete-access-key --user-name "$u" --access-key-id "$k"
      done
    done
```

`tr` plus `while read` rather than `USERS=$(...)`, because `--output text` returns the names
tab-separated on one line and **zsh does not word-split an unquoted variable**, so the shorter
form collapses all three into a single name on a default macOS shell.

Then the stack, with the delete guarded before the wait:

```shell
if aws cloudformation delete-stack --stack-name eks-project --region us-west-1; then
  aws cloudformation wait stack-delete-complete --stack-name eks-project --region us-west-1
else
  echo "delete-stack failed, so not waiting on it" >&2
fi
```

`wait` turns a failed deletion into a non-zero exit rather than something found months later, but
it is a bad way to learn the delete call itself was rejected: its waiter is `delay=30s` with
`maxAttempts=120`, so 60 minutes before exit 255, and no acceptor matches a stack left untouched
in `CREATE_COMPLETE`. The `if` reports that case immediately.

### Delete user password from parameter store

```shell
aws ssm delete-parameter --name EKSUserPassword --region us-west-1
```

### Delete the EC2 Key Pair

```shell
aws ec2 delete-key-pair --key-name EKSProjectEC2KeyPair --region us-west-1
```