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
kubectl create secret generic mysql-password --from-literal=password=this-is-my-password123cd@1 --namespace=dev-stateful-ebs 
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
mysql-password        Opaque                                1      85s
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

Go to `Volumes` in `AWS Console` https://us-west-1.console.aws.amazon.com/ec2/v2/home?region=us-west-1#Volumes: and delete the pvc (the ones to delete should all be in available state)

Then, run `kubectl get pv` and delete the following `pv`
* dev-stateful-ebs/mysql-pv-claim (name: pvc-88172a08-22f9-4523-8cf0-701ef88dd04d)   
* dev-stateful-ebs/wp-pv-claim (name: pvc-75f605cd-9b97-4315-ae46-0bfa13d4f5c2)

```shell
kubectl delete pv pvc-88172a08-22f9-4523-8cf0-701ef88dd04d

kubectl delete pv pvc-75f605cd-9b97-4315-ae46-0bfa13d4f5c2
```

Sample output:

```shell
persistentvolume "pvc-88172a08-22f9-4523-8cf0-701ef88dd04d" force deleted
```

At this point, if you check your pvs you will see they are in terminating status:

```shell
kubectl get pv 
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS        CLAIM                                    STORAGECLASS   REASON   AGE
pvc-28f1129c-ad1d-45df-8e52-8ab24c578ce5   8Gi        RWO            Delete           Bound         default/redis-data-redis-test-slave-0    gp2                     13d
pvc-4018b8ff-ff94-4ed9-8c5f-3148f77b7169   8Gi        RWO            Delete           Bound         default/redis-data-redis-test-master-0   gp2                     13d
pvc-75f605cd-9b97-4315-ae46-0bfa13d4f5c2   20Gi       RWO            Delete           Terminating   dev-stateful-ebs/wp-pv-claim             gp2                     6d22h
pvc-7d93df27-9460-4fb5-9790-884aba63237b   8Gi        RWO            Delete           Bound         default/redis-data-redis-test-slave-1    gp2                     13d
pvc-88172a08-22f9-4523-8cf0-701ef88dd04d   20Gi       RWO            Delete           Terminating   dev-stateful-ebs/mysql-pv-claim          gp2                     6d22h
```

Now, we are going to set the status to lost

```shell
kubectl patch pv pvc-88172a08-22f9-4523-8cf0-701ef88dd04d -p '{"metadata":{"finalizers":null}}'

kubectl patch pv pvc-75f605cd-9b97-4315-ae46-0bfa13d4f5c2 -p '{"metadata":{"finalizers":null}}'
```

### Delete secret

```shell
kubectl delete secret mysql-password
```

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

```shell
2022-08-09 10:02:07 [ℹ]  deleting EKS cluster "basic-eks-cluster"
2022-08-09 10:02:07 [ℹ]  deleted 0 Fargate profile(s)
2022-08-09 10:02:08 [✔]  kubeconfig has been updated
2022-08-09 10:02:08 [ℹ]  cleaning up AWS load balancers created by Kubernetes objects of Kind Service or Ingress
2022-08-09 10:02:09 [ℹ]  1 task: { delete cluster control plane "basic-eks-cluster" [async] }
2022-08-09 10:02:09 [ℹ]  will delete stack "eksctl-basic-eks-cluster-cluster"
2022-08-09 10:02:09 [✔]  all cluster resources were deleted
```

### Delete CFN tacks

```shell
aws cloudformation delete-stack --stack-name 	service-support
```

### Delete user password from parameter store

```shell
aws ssm delete-parameter --name EKSUserPassword --region us-west-1
```

### Delete the EC2 Key Pair

```shell
aws ec2 delete-key-pair --key-name EKSProjectEC2KeyPair --region us-west-1
```