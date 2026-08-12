# Deploying a Stateful App with EFS

We are deploying WordPress with MySQL Db using EFS to store data (of both).

This example consists of the following components:

* WP as frontend (with EBS to store data). WP pod will connect to MySQL pod thorugh kubernetes MySQL Service.
* MySQL as backend (with EFS to store data)
* A Load Balancer which is going to expose our app

All the WP pods (which are going to be in different AZ) are going to access to the `SAME EFS volume`

## Enable EFS

### Get VPC
If the VPC was not created in your default region, remember to pass the region flag with the proper region

```shell
aws ec2 describe-vpcs --region us-west-1
```

From the output, save the VPC ID. Example: `vpc-033362593e65e0f7a`

### Create a file system

```shell
aws efs create-file-system \
  --encrypted \
  --creation-token FileSystemForEKSProject \
  --tags Key=Name,Value=EKSwithEFS \
  --region us-west-1
```

Output:

```shell
{
    "OwnerId": "your-aws-account-d",
    "CreationToken": "FileSystemForEKSProject",
    "FileSystemId": "fs-00faa67859221a9e9",
    "FileSystemArn": "arn:aws:elasticfilesystem:us-west-1:your-aws-account-d:file-system/fs-00faa67859221a9e9",
    "CreationTime": "2022-08-09T09:27:56-07:00",
    "LifeCycleState": "creating",
    "Name": "EKSwithEFS",
    "NumberOfMountTargets": 0,
    "SizeInBytes": {
        "Value": 0,
        "ValueInIA": 0,
        "ValueInStandard": 0
    },
    "PerformanceMode": "generalPurpose",
    "Encrypted": true,
    "KmsKeyId": "arn:aws:kms:us-west-1:your-aws-account-d:key/147e3dce-dee0-4545-86bd-b7f8d02b9b23",
    "ThroughputMode": "bursting",
    "Tags": [
        {
            "Key": "Name",
            "Value": "EKSwithEFS"
        }
    ]
}
```

From the output, save the File System Id. Example: `fs-00faa67859221a9e9`

NOTE: Optionally you can `enable lifecycle management` (aws efs put-lifecycle-configuration...)

<!-- ### Create Security Group for the EC2 instances

You will need your VPC id

```shell
aws ec2 create-security-group \
--group-name EKSwithEFS-ec2-sg \
--description "EKSwithEFS, SG for EC2 instances" \
--vpc-id vpc-033362593e65e0f7a \
--region us-west-1
```

Output: 

```shell
{
    "GroupId": "sg-0c231baffb124d69d"
}
``` -->

<!-- ### Create Security Group for the Amazon EFS mount target

You will need your VPC id

```shell
aws ec2 create-security-group \
--group-name EKSwithEFS-mt-sg \
--description "EKSwithEFS, SG for mount target" \
--vpc-id vpc-033362593e65e0f7a \
--region us-west-1
```

Output:

```shell
{
    "GroupId": "sg-0c29da0ca7d5a4430"
}
``` -->

<!-- ### Add rules to Security Group

#### MT Security Group rule

Authorize inbound access to the security group for the Amazon EFS mount target.

```shell
aws ec2 authorize-security-group-ingress \
  --group-id ID of the security group created for Amazon EFS mount target \
  --protocol tcp \
  --port 2049 \
  --source-group ID of the security group created for EC2 instance \
  --region us-west-1
```

Example:

```shell
aws ec2 authorize-security-group-ingress \
  --group-id sg-0c29da0ca7d5a4430 \
  --protocol tcp \
  --port 2049 \
  --source-group sg-0b1be67f4b7c2d3b2\
  --region us-west-1 
```

Output:

```json
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0f98c2385c4dcbf1f",
            "GroupId": "sg-0c29da0ca7d5a4430",
            "GroupOwnerId": "your-aws-account-id",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 2049,
            "ToPort": 2049,
            "ReferencedGroupInfo": {
                "GroupId": "sg-0b1be67f4b7c2d3b2",
                "UserId": "your-aws-account-id"
            }
        }
    ]
}
```

#### EC2 Security Group rule

Authorize incoming Secure Shell (SSH) connections to the security group for our EC2 instance (efs-walkthrough1-ec2-sg) so we can connect to our EC2 instance using SSH from any host.

You will need the Security Group created for EC2 (example: sg-0b1be67f4b7c2d3b2)

```shell
aws ec2 authorize-security-group-ingress \
  --group-id sg-0b1be67f4b7c2d3b2 \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0 \
  --region us-west-1
```

Output:

```shell
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-01bd03702bc3093b8",
            "GroupId": "sg-0b1be67f4b7c2d3b2",
            "GroupOwnerId": "your-aws-account-id",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 22,
            "ToPort": 22,
            "CidrIpv4": "0.0.0.0/0"
        }
    ]
}
``` -->

### Create a mount target

We are going to need:
* File system id (example: fs-00faa67859221a9e9)
<!-- * Security Group for MT (example: sg-0c29da0ca7d5a4430 ) -->
* ClusterSharedNodeSecurityGroup (example: sg-0aa14bc28e6038b9a)
* Subnet Ids of all our nodes (examples: subnet-022fe0d51af2ab6f0 and subnet-0e3dc80d1dc559746)
These are the subnets used by our EC2s

```shell
aws efs create-mount-target \
  --file-system-id file-system-id \
  --subnet-id  subnet-id \
  --security-group ID-of-the security-group-created-for-mount-target \
  --region us-west-1
```

Example:

```shell
aws efs create-mount-target \
  --file-system-id fs-00faa67859221a9e9 \
  --subnet-id  subnet-022fe0d51af2ab6f0 \
  --security-group sg-0c29da0ca7d5a4430 \
  --region us-west-1

aws efs create-mount-target \
  --file-system-id fs-00faa67859221a9e9 \
  --subnet-id  subnet-0e3dc80d1dc559746 \
  --security-group sg-0c29da0ca7d5a4430 \
  --region us-west-1
```

After a few minutes, you can go to your File System in EFS and ccheck the Nertwork tab.
You should see 2 AZs in state `Available` for `Mount target state`

### Mount file system on EC2 instances

#### Change keypair pem permissions

```shell
chmod 400 ~/.ssh/EKSProjectEC2KeyPair.pem
```

#### SSH into our EC2s

We should do the following in ALL our instances (in our example, 2).

NOTE: If you are using an official AWS AMI the EFS client should come installed.

```shell
ssh -i "EKSProjectEC2KeyPair.pem" root@ec2-54-177-77-176.us-west-1.compute.amazonaws.com
```

**Update and reboot**

```shell
sudo yum -y update  
sudo reboot  
```

#### Install NFS client

```shell
sudo yum -y install nfs-utils
```

## Create namespace

```shell
kubectl create namespace dev-stateful-efs
```

Output:

```shell
namespace/dev-stateful-efs created
```

## Install aws-efs-csi-driver
This will enable communication between the storage and the underlying file system.

```shell
helm repo add aws-efs-csi-driver  https://kubernetes-sigs.github.io/aws-efs-csi-driver/
```

Output:

```shell
"aws-efs-csi-driver" has been added to your repositories
```

Then...

```shell
helm install aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver
```

Output:

```shell
W0811 08:23:39.945892   74799 warnings.go:70] spec.template.spec.nodeSelector[beta.kubernetes.io/os]: deprecated since v1.14; use "kubernetes.io/os" instead
NAME: aws-efs-csi-driver
LAST DEPLOYED: Thu Aug 11 08:23:38 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
To verify that aws-efs-csi-driver has started, run:

    kubectl get pod -n default -l "app.kubernetes.io/name=aws-efs-csi-driver,app.kubernetes.io/instance=aws-efs-csi-driver"
```

Execute:

```shell
kubectl get pod -n default -l "app.kubernetes.io/name=aws-efs-csi-driver,app.kubernetes.io/instance=aws-efs-csi-driver"
```

Sample output:

```shell
NAME                                 READY   STATUS    RESTARTS   AGE
efs-csi-controller-8cd49f9f6-h2wlm   3/3     Running   0          60s
efs-csi-controller-8cd49f9f6-tfk8m   3/3     Running   0          60s
efs-csi-node-8lzbs                   3/3     Running   0          60s
efs-csi-node-x72x2                   3/3     Running   0          60s
```

## Create storage class

```shell
kubectl apply -f eks/storageclass.yaml --namespace=dev-stateful-efs
```

Output:

```shell
storageclass.storage.k8s.io/efs-sc created
```

## Create persistent volume

First, replace `file-system-id` in `eks/pv.yaml ` with yours (example: fs-00faa67859221a9e9)

```shell
kubectl apply -f eks/pv.yaml --namespace=dev-stateful-efs
```

Output:

```shell
persistentvolume/efs-psv-eks created
```

### Optional: Get persistent volume

```shell
kubectl get pv -n dev-stateful-efs
```

Sample output:

```shell
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
efs-psv-eks   20Gi       RWX            Retain           Available           efs-sc                  114s
```

## Create persistent volume
This will bound our persistent volume

```shell
kubectl apply -f eks/pvc.yaml --namespace=dev-stateful-efs
```

Sample output:

```shell
persistentvolumeclaim/efs-pv-claim created
```

## Backend

### Create secret for MySQL password

```shell
kubectl create secret generic mysql-password --from-literal=password=$(openssl rand -base64 24) \
  --from-literal=wordpress-password=$(openssl rand -base64 24) --namespace=dev-stateful-efs
```

⚠️ **Only generate fresh values against an empty file system.** This variant retains its EFS
data (`persistentVolumeReclaimPolicy: Retain`), so on a second run through this walkthrough the
MySQL directory already exists — and **MySQL only applies `MYSQL_PASSWORD` when it initialises an
empty data directory**. Generate a new secret over retained data and the `wordpress` database
account keeps its *old* password while WordPress is handed the new one, so the app cannot connect
and neither side reports anything about a password mismatch.

Measured on `mysql:8.0`, reusing one volume across two starts:

```text
run 1, empty datadir,  MYSQL_PASSWORD=FIRSTPASS   -> wordpress user created with FIRSTPASS
run 2, same volume,    MYSQL_PASSWORD=SECONDPASS  -> ERROR 1045 (28000): Access denied
                                                     for user 'wordpress'@'localhost'
run 2, same volume,    old FIRSTPASS              -> still works
```

So on a re-run, pick one:

- **Keep the data**: do not recreate the secret. Leave the existing `mysql-password` in place, and
  if it is gone, recover the values rather than inventing them:

  ```shell
  kubectl get secret mysql-password --namespace=dev-stateful-efs \
    -o jsonpath='{.data.wordpress-password}' | base64 -d; echo
  ```

- **Start clean**: delete the EFS file system, or empty the `mysql/` directory on it, *before*
  creating a new secret. The [Clean up](#efs) section covers this, and note that deleting the PVC
  and PV does **not** remove the data.

The EBS variant does not have this problem in the same way: its volumes are `Delete` reclaim
policy, so a fresh run genuinely starts empty.

Output:

```shell
secret/mysql-password created
```

We can check the secrets for our namespace:

```shell
kubectl get secrets --namespace=dev-stateful-efs
```

Output:

```shell
NAME                  TYPE                                  DATA   AGE
default-token-fnzpq   kubernetes.io/service-account-token   3      32m
mysql-password        Opaque                                2      50s
```

### Create Service and Deployment for MySQL

```shell
kubectl apply -f eks/service-and-deployment-mysql.yaml --namespace=dev-stateful-efs 
```

Output:

```shell
service/wordpress-mysql created
deployment.apps/wordpress-mysql created
```

Check running pods in our namespace:

```shell
kubectl get pods -o wide -n dev-stateful-efs
```

Output:

```text
NAME                               READY   STATUS    RESTARTS   AGE     IP              NODE                                         NOMINATED NODE   READINESS GATES
wordpress-mysql-7d5dc78494-vvfxk   1/1     Running   0          7m29s   192.168.47.27   ip-192-168-53-8.us-west-1.compute.internal   <none>           <none>
```

You can go to EC2 Dashboard and search for the node `192-168-53-8` and ssh into it:

```shell
ssh -i "EKSProjectEC2KeyPair.pem" root@ec2-3-101-16-90.us-west-1.compute.amazonaws.com
```

Once connected:

```shell
sudo yum update

sudo mount | grep csi
```

Sample output:

```text
127.0.0.1:/ on /var/lib/kubelet/pods/414ce7a8-ade5-4e1b-a2ea-d7e7350e80d9/volumes/kubernetes.io~csi/efs-psv-eks/mount type nfs4 (rw,relatime,vers=4.1,rsize=1048576,wsize=1048576,namlen=255,hard,noresvport,proto=tcp,port=20079,timeo=600,retrans=2,sec=sys,clientaddr=127.0.0.1,local_lock=none,addr=127.0.0.1)
```

## Frontend

### Deploy via Deployment

#### Create Service and Deployment for WP

```shell
kubectl apply -f eks/service-and-deployment-wp.yaml --namespace=dev-stateful-efs
```

Output:

```shell
service/wordpress created
deployment.apps/wordpress created
```

And we can get the pods in our namespace:

```shell
kubectl get pods -n dev-stateful-efs
```

Sample output:

```shell
NAME                               READY   STATUS    RESTARTS   AGE
wordpress-77b7ff4d48-r878l         0/1     Pending   0          49s
wordpress-mysql-7d5dc78494-vvfxk   1/1     Running   0          16m
```

<!-- #### List persistent volumes for namespace

```shell
kubectl get pvc --namespace=dev-stateful-ebs
```

Sample ouput:

```shell
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-pv-claim   Bound    pvc-88172a08-22f9-4523-8cf0-701ef88dd04d   20Gi       RWO            gp2            6d22h
wp-pv-claim      Bound    pvc-75f605cd-9b97-4315-ae46-0bfa13d4f5c2   20Gi       RWO            gp2            6d22h
``` -->

Then, we can go to `Load Balancers` in `AWS Console`: https://us-west-1.console.aws.amazon.com/ec2/v2/home?region=us-west-1#LoadBalancers:sort=desc:createdTime
and use the DNS name (example: ae246843b31744420869aa7d2f24b679-1121499175.us-west-1.elb.amazonaws.com) to set up our `WP site`

## Clean up

### Frontend

```shell
kubectl delete -f eks/service-and-deployment-wp.yaml --namespace=dev-stateful-efs
```

Output:

```shell
service "wordpress" deleted
deployment.apps "wordpress" deleted
```

### Backend

```shell
kubectl delete -f eks/service-and-deployment-mysql.yaml --namespace=dev-stateful-efs
```

Output:

```shell
service "wordpress-mysql" deleted
deployment.apps "wordpress-mysql" deleted
```

### EFS

**Claim first, then the PV, and the file system outlives both.** This differs from the EBS
variant, and the difference is the reclaim policy: `efs/eks/pv.yaml` sets
`persistentVolumeReclaimPolicy: Retain` against a fixed `volumeHandle`. Under `Retain`,
deleting the claim moves the PV to `Released` rather than deleting it, and nothing touches the
data.

```shell
kubectl -n dev-stateful-efs delete pvc efs-pv-claim

kubectl delete pv efs-psv-eks
```

The `-n` is not optional for the claim: it lives in `dev-stateful-efs`, so without it the
command runs against your current context, reports `NotFound`, and the claim survives. The PV
takes no namespace, because a PersistentVolume is cluster-scoped.

**Deleting both leaves the data intact**, which is the part that surprises people. The file
system and everything in it survive, and because the PV names a fixed `volumeHandle`, re-applying
`efs/eks/pv.yaml` and the claim binds the same file system and the old WordPress install is
still there. So this step is required rather than tidying, and it is the only one that removes
the data:

Go to `EFS` in `AWS Console` https://us-west-1.console.aws.amazon.com/efs/home?region=us-west-1#/file-systems: and delete the filesystem (example: EKSwithEFS)

Do that **after** the Kubernetes objects, not before. Deleting the file system first leaves the
PV pointing at a `volumeHandle` that no longer exists.

<details>
<summary>What this section used to say, and why it left the PV stuck in Terminating</summary>

The original deleted the file system in the console first, then ran `kubectl delete pv` while
the claim still existed. A PV cannot finish deleting while a claim is bound to it, so it sat in
`Terminating`:

```text
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS        CLAIM                           STORAGECLASS   REASON   AGE
efs-psv-eks   20Gi       RWX            Retain           Terminating   dev-stateful-efs/efs-pv-claim   efs-sc                  24h
```

It then cleared the finalizer by hand, described as setting the status to lost:

```text
kubectl patch pv efs-psv-eks -p '{"metadata":{"finalizers":null}}'
```

That removes the API object without releasing anything, and it is only needed because the
deletion happened in the wrong order. Both blocks are quoted so the old instructions are
recognisable, not to be run.

</details>

**Not re-verified against EFS.** The corrected order is the documented behaviour of the
`Retain` reclaim policy, and this repository's verification ran on k3s with local-path storage,
not on EKS with EFS. See the README for what was and was not exercised.

### Delete secret

```shell
kubectl delete secret mysql-password --namespace=dev-stateful-efs
```

The secret was created with `--namespace=dev-stateful-efs`, so deleting it needs the same flag.
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