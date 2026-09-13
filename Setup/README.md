# EKS Playground Infrastructure Setup

This README is a quick rebuild guide for the Solar System EKS lab when
the AWS/KodeKloud Playground expires.

> **Important:** Playground AWS account IDs, VPC IDs, EKS endpoints, IAM
> ARNs, access keys, kubeconfig, NLB DNS names, and other generated
> values change between sessions. Do not blindly reuse values from an
> old Playground.

## Architecture

``` text
GitHub Actions
      |
      v
Amazon EKS
      |
      +-- Self-managed EC2 worker nodes
      |
      +-- ingress-nginx
              |
              v
AWS Load Balancer Controller
              |
              v
Internet-facing NLB
              |
              v
Ingress -> Service -> Solar System Pod :3000
```

## 1. Start the Playground and verify AWS identity

``` bash
aws sts get-caller-identity
aws configure get region
```

Use `us-east-1` throughout this lab.

Save the new AWS account ID:

``` bash
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo $AWS_ACCOUNT_ID
```

## 2. Create AWS access keys for GitHub Actions

Find the current Playground IAM username:

``` bash
aws sts get-caller-identity
```

Create an access key:

``` bash
aws iam create-access-key --user-name <YOUR_PLAYGROUND_USER>
```

Immediately save:

``` text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

Add/update them in the GitHub repository secrets.

## 3. Create the EKS cluster

In the AWS Console create an EKS cluster:

-   Name: `demo-eks`
-   Region: `us-east-1`
-   Auto Mode: OFF
-   Authentication mode: **EKS API and ConfigMap**
-   Use the Playground/default VPC
-   Select at least two suitable subnets
-   Cluster IAM role: role with `AmazonEKSClusterPolicy`

After the cluster is ACTIVE:

``` bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name demo-eks

kubectl get nodes
```

At this point no worker nodes may exist yet.

## 4. Create self-managed worker nodes

Managed Node Groups may be blocked by Playground permissions. Use the
current AWS EKS self-managed node CloudFormation template.

``` text
https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2025-11-26/amazon-eks-nodegroup.yaml
```

Before creating the stack, obtain the required cluster values:

``` bash
aws eks describe-cluster \
  --name demo-eks \
  --region us-east-1 \
  --query 'cluster.endpoint' \
  --output text

aws eks describe-cluster \
  --name demo-eks \
  --region us-east-1 \
  --query 'cluster.certificateAuthority.data' \
  --output text

aws eks describe-cluster \
  --name demo-eks \
  --region us-east-1 \
  --query 'cluster.kubernetesNetworkConfig.serviceIpv4Cidr' \
  --output text
```

For AL2023/nodeadm, do not leave the API server endpoint, certificate
authority, or service CIDR blank.

After CloudFormation creates the nodes, obtain the NodeInstanceRole ARN
and add it to `aws-auth`.

Example:

``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: <NODE_INSTANCE_ROLE_ARN>
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
```

Apply:

``` bash
kubectl apply -f aws-auth.yaml
kubectl get nodes -w
```

Wait for the nodes to become `Ready`.

## 5. Create application namespaces

``` bash
kubectl create namespace development --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace production --dry-run=client -o yaml | kubectl apply -f -
```

## 6. Install ingress-nginx

Install the ingress-nginx controller:

``` bash
kubectl apply -f \
https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/cloud/deploy.yaml
```

Verify:

``` bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

The `ingress-nginx-controller` Service should be type `LoadBalancer`.

## 7. Install eksctl

CloudShell may not include `eksctl`.

``` bash
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO \
"https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_${PLATFORM}.tar.gz"

tar -xzf eksctl_${PLATFORM}.tar.gz

mkdir -p $HOME/bin
mv eksctl $HOME/bin/

export PATH=$HOME/bin:$PATH

eksctl version
```

If needed after reopening the shell:

``` bash
export PATH=$HOME/bin:$PATH
```

## 8. Create the AWS Load Balancer Controller IAM policy

Download the controller IAM policy:

``` bash
curl -O \
https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.1/docs/install/iam_policy.json
```

Create the policy:

``` bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

Save the returned policy ARN.

You can also construct it using the current account ID:

``` bash
LBC_POLICY_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy"
echo $LBC_POLICY_ARN
```

## 9. Associate an IAM OIDC provider with EKS

``` bash
eksctl utils associate-iam-oidc-provider \
  --cluster demo-eks \
  --region us-east-1 \
  --approve
```

Verify:

``` bash
oidc_id=$(aws eks describe-cluster \
  --name demo-eks \
  --region us-east-1 \
  --query "cluster.identity.oidc.issuer" \
  --output text | cut -d '/' -f 5)

echo $oidc_id

aws iam list-open-id-connect-providers | grep $oidc_id
```

The second command should return an IAM OIDC provider ARN.

## 10. Create the Load Balancer Controller IAM ServiceAccount

``` bash
eksctl create iamserviceaccount \
  --cluster=demo-eks \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn="$LBC_POLICY_ARN" \
  --override-existing-serviceaccounts \
  --region=us-east-1 \
  --approve
```

Verify:

``` bash
kubectl get serviceaccount aws-load-balancer-controller \
  -n kube-system \
  -o yaml
```

Look for:

``` yaml
eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/AmazonEKSLoadBalancerControllerRole
```

## 11. Install Helm

CloudShell may not include Helm.

``` bash
curl -fsSL \
https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm version
```

Add the AWS EKS charts repository:

``` bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

## 12. Install AWS Load Balancer Controller

First get the current Playground VPC ID:

``` bash
VPC_ID=$(aws eks describe-cluster \
  --name demo-eks \
  --region us-east-1 \
  --query "cluster.resourcesVpcConfig.vpcId" \
  --output text)

echo $VPC_ID
```

Install the controller and explicitly pass the region and VPC ID.

This is important in the Playground because the controller may be unable
to discover the VPC through EC2 Instance Metadata.

``` bash
helm install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-eks \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId="$VPC_ID"
```

If it was already installed without `region`/`vpcId`, fix it with:

``` bash
helm upgrade aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-eks \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId="$VPC_ID"
```

Verify:

``` bash
kubectl rollout status \
  deployment/aws-load-balancer-controller \
  -n kube-system

kubectl get pods -n kube-system \
  -l app.kubernetes.io/name=aws-load-balancer-controller
```

Expected: controller pods are `1/1 Running`.

### CrashLoopBackOff troubleshooting

If the logs show:

``` text
failed to get VPC ID
failed to fetch VPC ID from instance metadata
context deadline exceeded
```

the `region` and `vpcId` Helm values above are the fix.

Inspect the previous crashed container with:

``` bash
kubectl logs -n kube-system \
  deployment/aws-load-balancer-controller \
  --all-containers=true \
  --previous
```

## 13. Make ingress-nginx use an internet-facing NLB

Annotate the ingress-nginx controller Service:

``` bash
kubectl annotate service ingress-nginx-controller \
  -n ingress-nginx \
  service.beta.kubernetes.io/aws-load-balancer-type=external \
  service.beta.kubernetes.io/aws-load-balancer-nlb-target-type=instance \
  --overwrite
```

``` bash
kubectl annotate service ingress-nginx-controller \
  -n ingress-nginx \
  service.beta.kubernetes.io/aws-load-balancer-scheme=internet-facing \
  --overwrite
```

Watch the Service:

``` bash
kubectl get svc -n ingress-nginx ingress-nginx-controller -w
```

Check AWS:

``` bash
aws elbv2 describe-load-balancers \
  --region us-east-1 \
  --query 'LoadBalancers[*].[LoadBalancerName,DNSName,State.Code,Type]' \
  --output table
```

Wait until the NLB state becomes:

``` text
active
```

The type should be:

``` text
network
```

and the scheme should be `internet-facing`.

## 14. Verify DNS and external application access

Get the NLB hostname:

``` bash
NLB_HOST=$(kubectl get svc ingress-nginx-controller \
  -n ingress-nginx \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

echo $NLB_HOST
```

Test DNS:

``` bash
nslookup "$NLB_HOST"
```

Test the application:

``` bash
curl -v "http://$NLB_HOST/live"
```

Expected:

``` json
{"status":"live"}
```

Verify the application Ingress also contains the NLB address:

``` bash
kubectl get ingress -n development
```

## 15. GitHub Actions kubeconfig

Generate the kubeconfig for the new Playground:

``` bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name demo-eks
```

Display it:

``` bash
cat ~/.kube/config
```

Update the GitHub repository secret:

``` text
KUBE_CONFIG
```

Do this every time the Playground is rebuilt because the account/cluster
endpoint can change.

## 16. GitHub repository secrets and variables

Update the values required by the workflow.

Typical secrets:

``` text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
KUBE_CONFIG
MONGO_PASSWORD
AWS_S3_BUCKET
SLACK_WEBHOOK
```

Typical variables:

``` text
DOCKERHUB_USERNAME
MONGO_URI
MONGO_USERNAME
```

Do not store the generated NLB hostname as a permanent GitHub secret.
The deployment workflow should read it from Kubernetes.

Example:

``` bash
APP_INGRESS_HOST=$(kubectl -n "$NAMESPACE" get ingress solar-system \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
```

## 17. Create the S3 bucket used by GitHub Actions

Choose a globally unique bucket name:

``` bash
aws s3api create-bucket \
  --bucket <UNIQUE_BUCKET_NAME> \
  --region us-east-1
```

Verify:

``` bash
aws s3 ls
```

Set the GitHub secret `AWS_S3_BUCKET` to the bucket **name**, not the
ARN.

## 18. Final validation

Run:

``` bash
kubectl get nodes
kubectl get pods -A
kubectl get svc -A
kubectl get ingress -A
```

Then:

``` bash
aws elbv2 describe-load-balancers \
  --region us-east-1 \
  --query 'LoadBalancers[*].[DNSName,State.Code,Type]' \
  --output table
```

And:

``` bash
NLB_HOST=$(kubectl get svc ingress-nginx-controller \
  -n ingress-nginx \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

nslookup "$NLB_HOST"
curl "http://$NLB_HOST/live"
```

Expected final response:

``` json
{"status":"live"}
```

Then rerun the GitHub Actions workflow.

## Quick troubleshooting

### `eksctl: command not found`

Install `eksctl` using section 7.

### `helm: command not found`

Install Helm using section 11.

### LoadBalancer hostname exists but DNS returns NXDOMAIN

Check whether AWS actually has a Load Balancer:

``` bash
aws elb describe-load-balancers --region us-east-1
aws elbv2 describe-load-balancers --region us-east-1
```

If both are empty, the Kubernetes Service status may contain a
stale/legacy hostname. Install/configure the AWS Load Balancer
Controller and provision the NLB as described above.

### AWS Load Balancer Controller is CrashLoopBackOff

``` bash
kubectl logs -n kube-system \
  deployment/aws-load-balancer-controller \
  --all-containers=true \
  --previous
```

If VPC discovery through instance metadata fails, pass `region` and
`vpcId` explicitly using `helm upgrade`.

### Pod is healthy but external URL fails

Test the application internally:

``` bash
kubectl get pods -n development -o wide
kubectl get endpoints -n development solar-system
```

Then:

``` bash
kubectl run curl-test \
  -n development \
  --image=curlimages/curl \
  --rm -it \
  --restart=Never \
  -- curl -v http://solar-system:3000/live
```

If this returns `{"status":"live"}`, the Pod and Service are healthy;
continue troubleshooting the Ingress/NLB layer.

### GitHub Actions uses an old cluster

Regenerate kubeconfig:

``` bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name demo-eks
```

Then replace the `KUBE_CONFIG` GitHub secret.

------------------------------------------------------------------------

## Rebuild checklist

``` text
[ ] Start new AWS Playground
[ ] Verify AWS account/region
[ ] Create new AWS access keys
[ ] Create demo-eks
[ ] Create self-managed worker nodes
[ ] Apply aws-auth
[ ] Wait for nodes Ready
[ ] Create development/production namespaces
[ ] Install ingress-nginx
[ ] Install eksctl
[ ] Create AWS Load Balancer Controller IAM policy
[ ] Associate EKS OIDC provider
[ ] Create controller IAM ServiceAccount/role
[ ] Install Helm
[ ] Get current VPC ID
[ ] Install AWS Load Balancer Controller with region + vpcId
[ ] Verify controller pods Running
[ ] Annotate ingress-nginx Service for external NLB
[ ] Wait for NLB active
[ ] Test nslookup
[ ] Test /live
[ ] Regenerate kubeconfig
[ ] Update GitHub secrets/variables
[ ] Create/update S3 bucket
[ ] Rerun GitHub Actions
```

## Notes

The Playground is ephemeral. The following values should always be
treated as temporary:

``` text
AWS account ID
AWS access keys
EKS endpoint
kubeconfig
VPC ID
IAM role/policy ARNs
NodeInstanceRole ARN
NLB hostname
EC2 instance IDs/IPs
```

The goal of this README is to make rebuilding the environment repeatable
instead of preserving old generated values.
