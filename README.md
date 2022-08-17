# Deploying a Stateful App

<!-- TODO:
  Differences between EBS and EFS
-->

Note: 
* EBS is tied to a particular AZ.
* EFS can be accessed from different AZ through a Security Group.

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

