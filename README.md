# Kube2IAM Setup for Kubernetes on EC2

This repository contains the necessary Kubernetes manifests to set up kube2iam, which allows pods to assume IAM roles without storing AWS credentials in the cluster.

## What is Kube2IAM?

Kube2IAM is a daemon that provides IAM credentials to containers running inside a Kubernetes cluster based on annotations. It intercepts AWS API calls and provides temporary credentials for the specified IAM role.

## Prerequisites

- Kubernetes cluster running on AWS EC2 instances
- kubectl configured to access your cluster

## Required IAM Roles

### 1. EC2 Instance Role (for kube2iam daemon)

The EC2 instances running your Kubernetes cluster need an IAM role with the following policy:

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
Instead of *arn:aws:iam::585768175989:role/aws-cli-kube2iam* you add any role you want your pods will assume

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

1. **Create EC2 Instance Role:**
   ```bash
   aws iam create-role --role-name EC2-Kube2IAM-Role --assume-role-policy-document file://ec2-trust-policy.json
   aws iam attach-role-policy --role-name EC2-Kube2IAM-Role --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess
   aws iam put-role-policy --role-name EC2-Kube2IAM-Role --policy-name AssumeRolePolicy --policy-document file://assume-role-policy.json
   ```

2. **Create Application Role:**
   ```bash
   aws iam create-role --role-name aws-cli-kube2iam --assume-role-policy-document file://app-trust-policy.json
   aws iam put-role-policy --role-name aws-cli-kube2iam --policy-name AppPermissions --policy-document file://app-permissions.json
   ```

3. **Attach EC2 Instance Role to your EC2 instances:**
   ```bash
   aws ec2 associate-iam-instance-profile --instance-id i-1234567890abcdef0 --iam-instance-profile Name=EC2-Kube2IAM-InstanceProfile
   ```

### Step 2: Deploy Kube2IAM

1. **Deploy the ServiceAccount and RBAC:**
   ```bash
   kubectl apply -f service-account.yaml
   ```

2. **Deploy the Kube2IAM DaemonSet:**
   ```bash
   kubectl apply -f deamon-set.yaml
   ```

3. **Verify Kube2IAM is running:**
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
   kubectl exec -it deployment/aws-iam-tester -- aws sts get-caller-identity
   ```

## Configuration Details

### Kube2IAM DaemonSet Configuration

- **Image:** `jtblin/kube2iam:latest`
- **Port:** 8181 (HTTP API)
- **Host Network:** Enabled for iptables rules
- **Tolerations:** Allows scheduling on nodes with `kube2iam` taint
- **Auto-discovery:** Automatically discovers base ARN and default role

### Application Deployment Configuration

- **Image:** `amazon/aws-cli`
- **IAM Role:** `arn:aws:iam::585768175989:role/aws-cli-kube2iam`
- **Region:** `il-central-1`
- **Tolerations:** Required to run on nodes with kube2iam taint

## Troubleshooting

### Check Kube2IAM Logs
```bash
kubectl logs -n kube-system -l name=kube2iam
```

### Verify IAM Role Assumption
```bash
kubectl exec -it deployment/aws-iam-tester -- aws sts get-caller-identity
```

### Check Pod Annotations
```bash
kubectl describe pod -l app=aws-iam-tester
```

### Common Issues

1. **403 Forbidden:** Check if the EC2 instance role has `sts:AssumeRole` permission
2. **Role not found:** Verify the IAM role ARN in the pod annotation
3. **Pod not starting:** Check if the node has the required tolerations

## Security Considerations

- Only grant necessary permissions to application roles
- Use least privilege principle
- Regularly rotate and audit IAM roles
- Monitor kube2iam logs for unauthorized access attempts

## Files Overview

- `service-account.yaml` - ServiceAccount and RBAC for kube2iam
- `deamon-set.yaml` - Kube2IAM daemon deployment
- `deployment.yaml` - Example application using IAM role assumption

## Additional Resources

- [Kube2IAM GitHub Repository](https://github.com/jtblin/kube2iam)
- [AWS IAM Roles Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)
- [Kubernetes Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
