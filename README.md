# Kube2IAM Setup for Kubernetes Cluster on EC2

This repository contains the necessary Kubernetes manifests to set up kube2iam, which allows pods to assume IAM roles without storing AWS credentials in the cluster.

**Note:** This setup is designed to work with IMDSv2 Token Required and http-hop-limit=1 for enhanced security.

## What is Kube2IAM?

Kube2IAM is a daemon that provides IAM credentials to containers running inside a Kubernetes cluster based on annotations. It intercepts AWS API calls and provides temporary credentials for the specified IAM role.

## Prerequisites

- Kubernetes cluster running on AWS EC2 instances
- kubectl configured to access your cluster
- AWS CLI configured on your local machine
- EC2 instances configured with IMDSv2 (Token Required, http-hop-limit=1)

## Required IAM Roles

### 1. EC2 Instance Role (for kube2iam daemon)

The EC2 instances (Worker Nodes) running your Kubernetes cluster need an IAM role with the following policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::585768175989:role/aws-cli-kube2iam" 
        }
    ]
}
```
**Note:** Replace or add to `arn:aws:iam::585768175989:role/aws-cli-kube2iam` with any role you want your pods to assume.

**Trust Policy for EC2 Instance Role:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

### 2. Application Role (for your pods)

Create an IAM role that your application pods will assume. In this example, we use `aws-cli-kube2iam` role.

**Trust Policy for Application Role:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::585768175989:role/kubernetes-worker-role"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

**Permissions Policy for Application Role:**
Add the AWS services and actions your application needs. For example:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:ListBucket"
            ],
            "Resource": "*"
        }
    ]
}
```

## Setup Instructions

### Step 1: Create IAM Roles

1. **Create EC2 Instance Role (kubernetes-worker-role)**



2. **Create Application Role(aws-cli-kube2iam) and any Role you need for each workload**


3. **Attach EC2 Instance Role to your EC2 instances:**
   ```bash
   aws ec2 associate-iam-instance-profile --instance-id i-1234567890abcdef0 --iam-instance-profile Name=kubernetes-worker-node
   ```

### Step 2: Deploy Kube2IAM

1. **Deploy the ServiceAccount and RBAC**
   ```bash
   kubectl apply -f service-account.yaml
   ```

2. **Deploy the Kube2IAM DaemonSet**
   ```bash
   kubectl apply -f deamon-set.yaml
   ```

3. **Verify Kube2IAM is running**
   ```bash
   kubectl get pods -n kube-system | grep kube2iam
   ```

### Step 3: Deploy Your Application

1. **Deploy the test application:**
   ```bash
   kubectl apply -f deployment.yaml
   ```

2. **Test the IAM role assumption:**
   ```bash
   kubectl exec -it deployment/aws-iam-tester -- /bin/sh
   aws sts get-caller-identity
   ```

   You should get output like:
   ```json
   {
       "UserId": "AROAYQYUBBF243CTHO32Y:120348f1-aws-cli-kube2iam",
       "Account": "585768175989",
       "Arn": "arn:aws:sts::585768175989:assumed-role/aws-cli-kube2iam/120348f1-aws-cli-kube2iam"
   }
   ```

## Additional Resources

- [Kube2IAM GitHub Repository](https://github.com/jtblin/kube2iam)
