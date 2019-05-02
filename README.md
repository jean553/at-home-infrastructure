# at-home-infrastructure

The server part of [AtHome](https://github.com/jean553/at-home).

This documentation explains how to build and run the at-home infrastructure.

It is mainly based on these two official documentations:
 * https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html,
 * https://aws.amazon.com/blogs/opensource/kubernetes-ingress-aws-alb-ingress-controller/
(but contains the AtHome app specificities)

Most of these steps have to be done with the AWS CLI command tool and the AWS console.

## Requirements
 * awscli 1.16.147 minimum (https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)
 * kubectl (https://kubernetes.io/docs/tasks/tools/install-kubectl/, can be installed from Linux distros repos)
 * aws-iam-authenticator (https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html)

## Install the infrastructure

### Create the AtHome ECR

Docker images of the server will be stored on this private ECR.

Call the ECR `at-home-server`.

### Create the AtHome EKS IAM role

An IAM role is required to let EKS create AWS resources.

Create an IAM role called `at-home-eks-role` and **attach the EKS service** to this role.

### Create the AtHome EKS IAM user

An IAM user is required to let you execute `kubectl` and `awscli` commands.

Create an IAM user called `at-home-eks-user` with **programmatic access**.

Attach to him the policy `AmazonEC2ContainerRegistryFullAccess`.

Create and attach to him the two following policies:

#### EKSFullAccess

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "eks:*"
            ],
            "Resource": "*"
        }
    ]
}
```

#### PassRole

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:aws:iam::YOUR_CLIENT_ID:role/at-home-eks-role"
        }
    ]
}
```

### Create the AtHome EKS cluster VPC

Using **Cloud Formation**, create the cluster VPC stack called `at-home-eks-vpc` using the following AWS S3 template URL:

```
https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-02-11/amazon-eks-vpc-sample.yaml
```

Use the following network configuration:
 * VPC private CIDR range: `192.168.0.0/16`
 * Subnet 1: `192.168.64.0/18`
 * Subnet 2: `192.168.128.0/18`
 * Subnet 3: `192.168.192.0/18`

At this point, a VPC is ready, with subnets, routes and gateway all configured.

### Create the AtHome EKS cluster

Ensure your AWS local configuration credentials are those for the IAM user `at-home-eks-user`.

Create the cluster with the following command (and your EKS VPC subnets ids and security group id):

```sh
aws eks \
    --region eu-west-3 \
    create-cluster \
    --name at-home-eks-cluster \
    --role-arn arn:aws:iam::YOUR_CLIENT_ID:role/at-home-eks-role \
    --resources-vpc-config subnetIds=FIRST_SUBNET_ID,SECOND_SUBNET_ID,THIRD_SUBNET_ID,securityGroupIds=SECURITY_GROUP_ID
```

The `at-home-eks-cluster` is now creating.

Check the cluster status with:

```sh
aws eks \
    --region eu-west-3 \
    describe-cluster \
    --name at-home-eks-cluster \
    --query cluster.status
```

Wait until the returned status is `ACTIVE`.

At this point, the EKS cluster is created but not yet configured and launched.

### Download your kubectl configuration

Configure `kubectl` by directly downloading and saving at the right place the `kubectl` configuration from your cluster.

```sh
aws eks \
    --region eu-west-3 \
    update-kubeconfig \
    --name at-home-eks-cluster
```

### Create the AtHome EKS cluster nodes

Using **Cloud Formation**, create the stack called `at-home-eks-nodes` containing the cluster EC2 instances from the following AWS S3 template URL:

```
https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-02-11/amazon-eks-nodegroup.yaml
```

Use the following network configuration:
 * Cluster name: `at-home-eks-cluster`,
 * Security group: the one from `at-home-eks-vpc`,
 * Node group: `at-home-eks-node-group`,
 * ASG minimum size: 1,
 * ASG desired size: 3,
 * ASG maximum size: 4,
 * Instances type: `t3.medium`,
 * Image ID: `ami-0560aea042fec8b12` (EKS-optimized AMI for Paris region),
 * Nodes key: a SSH key you have to create,
 * VPC ID: `at-home-eks-vpc`,
 * Subnets: select all the `at-home-eks-vpc` subnets

At this point, the EC2 machines of the cluster are running.

### Include nodes into the cluster

For now, kubectl can access the cluster but the previously created nodes are still not part of the cluster yet.

Create and save the following configuration file as `at-home-eks-kubectl.yaml` and replace the value `CLUSTER_NODE_STACK_ROLE_ARN` by the ID of the role that has been created during the CloudFormation cluster nodes stack creation.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: CLUSTER_NODE_STACK_ROLE_ARN
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
```

Apply the configuration into `kubectl`:

```sh
kubectl apply -f at-home-eks-kubectl.yaml
```

The nodes are now part of the cluster.

## Run the service

Once the infrastructure is setup, you can now start the service.

Ensure there is at least one version of the `at-home-server` image on your ECR.

### Start a pod

```sh
kubectl run at-home-server --image=YOUR_CLIENT_ID.dkr.ecr.eu-west-3.amazonaws.com/at-home-server:latest
```

### Test the pod

Forward ports:

```sh
kubectl port-forward POD_NAME 8000:8000
```

Call the API:

```sh
curl http://localhost:8000/api/ping
```

## Configure the public access

This section explains how to access the service from `at-home.YOUR_DOMAIN.com`.

### Configure subnet tags for ingress controller auto discovery

Tag the three VPC subnets in order to let in the ingress controller automatically discover them.

Add the following tags to your subnets:
 * `kubernetes.io/role/internal-elb` with a value of 1,
 * `kubernetes.io/role/elb` with a value of 1,

### Give appropriate rights to the nodes group for ingress traffic handling

Create IAM policy called `IngressControllerPermission` from the JSON available here:

```sh
https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.0.0/docs/examples/iam-policy.json
```

Attach this policy to the role created during the nodes group stack creation (the role should have the format `at-home-eks-nodes-NodeInstanceRole-*`).

Apply the required roles and permissions required by the ALB ingress controller:

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.0.0/docs/examples/rbac-role.yaml
```

### Create the ALB ingress controller

Download the ALB ingress controller YAML from the following address:

```
https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.0.0/docs/examples/alb-ingress-controller.yaml
```

Add the two following lines (or modify them if they already exist) into the `spec` > `containers` > `args` section:

```
--cluster-name=at-home-eks-cluster
--feature-gates=waf=false
```

(AWS WAF firewall is disabled here because the feature does not exist in the deployment region eu-west-3)

Deploy the ALB ingress controller:

```sh
kubectl apply -f alb-ingress-controller.yaml
```

## Deploy the application

Deploy the namespace for all the AtHome kubernetes components:

```sh
kubectl apply -f at-home-server-namespace.yaml
```

Deploy the pods from the Docker image (set the correct ECR URL into the yaml configuration file first):

```sh
kubectl apply -f at-home-server-deployment.yaml
```

Deploy the service to expose all the pods:

```sh
kubectl apply -f at-home-server-service.yaml
```

Deploy the ingress controller to access the service from the Internet:

```sh
kubectl apply -f at-home-server-ingress.yaml
```

## Access the application from the Internet

Get the ALB DNS name:

```sh
kubectl get ingress --all-namespaces
```

The DNS address can now be used with Route53.
