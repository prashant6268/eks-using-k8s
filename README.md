# AWS EKS + Kubernetes Practical Project

A step-by-step guide to the EKS project built on an EC2 Bastion Host. It
covers Kubernetes access, Deployments, Services, the AWS Load Balancer
Controller, Ingress, and serving a custom HTML page.

> **Example environment used in this project** - AWS Region:
> `us-east-1` - EKS cluster: `k8s-2` - Kubernetes namespace: `default` -
> Application: Nginx - Deployment: `nginx-deployment` - Service:
> `nginx-service` - Ingress: `nginx-ingress` - AWS Load Balancer
> Controller namespace: `kube-system`
>
> Replace these example values with your own environment. Do not commit
> AWS credentials, tokens, private keys, or other secrets.

------------------------------------------------------------------------

## 1. Architecture overview

``` text
User / Internet
      |
      v
AWS Application Load Balancer (ALB)
      |
      v
Kubernetes Ingress (nginx-ingress)
      |
      v
Kubernetes Service (nginx-service:80)
      |
      v
Deployment (nginx-deployment)
      |
      +---- Nginx Pod 1
      +---- Nginx Pod 2
      +---- Nginx Pod 3
```

The AWS Load Balancer Controller watches Kubernetes Ingress resources
and provisions/configures an AWS Application Load Balancer. The ALB
sends traffic to the Kubernetes Service's Pod targets.

### Main Kubernetes objects

  ---------------------------------------------------------------------
  Object                             Purpose
  ---------------------------------- ----------------------------------
  Cluster                            Control plane and worker-node
                                     environment for running workloads

  Node                               Machine that runs Kubernetes Pods

  Pod                                Smallest deployable Kubernetes
                                     unit; contains one or more
                                     containers

  Deployment                         Maintains the desired number of
                                     Pod replicas and manages rolling
                                     updates

  ReplicaSet                         Keeps the requested number of Pod
                                     replicas running; usually managed
                                     by a Deployment

  Service                            Stable network endpoint for a
                                     changing set of Pods

  Ingress                            HTTP/HTTPS routing rules into the
                                     cluster

  ConfigMap                          Stores non-secret configuration,
                                     such as a custom HTML file

  ServiceAccount                     Kubernetes identity used by Pods

  OIDC / IRSA                        Lets a Kubernetes ServiceAccount
                                     obtain temporary AWS credentials
                                     through an IAM role
  ---------------------------------------------------------------------

------------------------------------------------------------------------

## 2. Prerequisites

You need:

-   An AWS account and permissions to administer the EKS cluster and
    required IAM resources.
-   An EKS cluster and a reachable EC2 Bastion Host.
-   AWS CLI configured with an intended IAM identity.
-   `kubectl`, `eksctl`, `helm`, `curl`, `tar`, and (optionally) `jq`.
-   Network access from the Bastion Host to the EKS API endpoint on TCP
    443. 
-   For a public ALB, suitable public subnets, route tables, tags, and
    security-group rules.

Check machine architecture:

``` bash
uname -m
```

For an x86_64 EC2 instance, the output should be `x86_64`.

Check AWS CLI identity:

``` bash
aws sts get-caller-identity
aws --version
```

**Security note:** Prefer an EC2 instance profile / IAM role for the
Bastion Host instead of long-lived IAM user access keys. Do not paste
credentials into this README.

------------------------------------------------------------------------

## 3. Install kubectl (Linux x86_64)

Download a kubectl version compatible with your EKS Kubernetes version.
The URL below is an example version; use the version recommended for
your cluster.

``` bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo install -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

------------------------------------------------------------------------

## 4. Install eksctl (Linux x86_64)

``` bash
curl --silent --location \
  "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" \
  --output eksctl.tar.gz

tar -xzf eksctl.tar.gz
sudo install -m 0755 eksctl /usr/local/bin/eksctl
eksctl version
```

Do not add the `eksctl` executable or its downloaded archive to Git. Add
them to `.gitignore`.

------------------------------------------------------------------------

## 5. Connect kubectl to the EKS cluster

Update the local kubeconfig:

``` bash
aws eks update-kubeconfig --region us-east-1 --name k8s-2
```

Check the current context and cluster:

``` bash
kubectl config current-context
kubectl get nodes
kubectl get namespaces
```

If `kubectl get nodes` times out, investigate network connectivity
first:

-   Confirm the EKS endpoint is reachable from the Bastion Host.
-   For a private endpoint, the Bastion Host must be in a connected
    network with correct routing.
-   Check the EKS cluster security group, Bastion security group,
    network ACLs, and routes.
-   TCP 443 must be allowed from the Bastion Host to the EKS API
    endpoint.

Useful checks:

``` bash
aws eks describe-cluster \
  --name k8s-2 \
  --region us-east-1 \
  --query 'cluster.resourcesVpcConfig'

aws eks describe-cluster \
  --name k8s-2 \
  --region us-east-1 \
  --query 'cluster.endpoint' \
  --output text
```

A timeout is a **network** problem; `Unauthorized` or `Forbidden` is an
**authentication/authorization** problem. Diagnose them separately.

### EKS access entries

An EKS access entry grants a specified IAM principal access to the
Kubernetes API. The IAM principal that runs `kubectl` must match the
principal authorized in EKS.

Check the active identity:

``` bash
aws sts get-caller-identity
```

List access entries:

``` bash
aws eks list-access-entries \
  --cluster-name k8s-2 \
  --region us-east-1
```

Example: create an access entry for an IAM user (use the actual ARN
returned by `get-caller-identity`; do not copy this example blindly):

``` bash
aws eks create-access-entry \
  --cluster-name k8s-2 \
  --region us-east-1 \
  --principal-arn arn:aws:iam::<ACCOUNT_ID>:user/<IAM_USER> \
  --type STANDARD
```

Grant cluster administrator access only when genuinely required and only
to a trusted principal:

``` bash
aws eks associate-access-policy \
  --cluster-name k8s-2 \
  --region us-east-1 \
  --principal-arn arn:aws:iam::<ACCOUNT_ID>:user/<IAM_USER> \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```

For production, use least-privilege access rather than giving every
operator cluster-admin permissions. Prefer an IAM role for the Bastion
Host and authorize that role in EKS.

Check associated policies:

``` bash
aws eks list-associated-access-policies \
  --cluster-name k8s-2 \
  --region us-east-1 \
  --principal-arn arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
```

------------------------------------------------------------------------

## 6. Deploy Nginx with a Deployment

Create `deployment.yaml`:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx-container
          image: nginx:latest
          ports:
            - containerPort: 80
```

Apply and inspect:

``` bash
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get replicasets
kubectl get pods
kubectl get pods -o wide
kubectl describe deployment nginx-deployment
```

The Deployment requests three replicas. Its selector `app: nginx` must
match the Pod template label `app: nginx`.

Scale the Deployment:

``` bash
kubectl scale deployment nginx-deployment --replicas=5
kubectl get pods
```

Roll out a restart:

``` bash
kubectl rollout restart deployment nginx-deployment
kubectl rollout status deployment nginx-deployment
```

Delete the Deployment when you no longer need it:

``` bash
kubectl delete deployment nginx-deployment
```

------------------------------------------------------------------------

## 7. Expose Nginx with a ClusterIP Service

A ClusterIP Service provides a stable internal endpoint for the Pods. It
is the normal backend Service type when using an ALB Ingress.

Create `service.yaml`:

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Apply and verify:

``` bash
kubectl apply -f service.yaml
kubectl get services
kubectl describe service nginx-service
kubectl get endpoints nginx-service
```

The Service selector `app: nginx` must match the labels on the
Deployment's Pods. If the Endpoints list is empty, check that the Pods
are Ready and the labels match.

### Optional: temporary port-forward test

``` bash
kubectl port-forward service/nginx-service 9090:80
```

Then, from another terminal on the Bastion Host:

``` bash
curl http://127.0.0.1:9090
```

Stop port-forwarding with `Ctrl+C`.

------------------------------------------------------------------------

## 8. Install Helm

Install Helm using the official Helm installation instructions for your
Linux distribution. Verify:

``` bash
helm version
```

Add and update the AWS EKS chart repository:

``` bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

------------------------------------------------------------------------

## 9. Set up OIDC and IAM Roles for Service Accounts (IRSA)

OIDC establishes trust between the EKS cluster's identity provider and
AWS IAM. IRSA lets a Kubernetes ServiceAccount assume an IAM role and
receive temporary AWS credentials. This avoids storing permanent AWS
access keys in Pods.

Associate the cluster OIDC provider:

``` bash
eksctl utils associate-iam-oidc-provider \
  --cluster k8s-2 \
  --region us-east-1 \
  --approve
```

Get the issuer URL:

``` bash
aws eks describe-cluster \
  --name k8s-2 \
  --region us-east-1 \
  --query 'cluster.identity.oidc.issuer' \
  --output text
```

List IAM OIDC providers:

``` bash
aws iam list-open-id-connect-providers
```

------------------------------------------------------------------------

## 10. Install the AWS Load Balancer Controller

The AWS Load Balancer Controller watches Kubernetes Ingress and Service
resources and creates/manages AWS load balancers.

### 10.1 Download the controller IAM policy

Use the policy file for the controller version you are installing. Check
the official AWS Load Balancer Controller documentation for the current
version and policy.

Example:

``` bash
curl -o iam_policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.1/docs/install/iam_policy.json
```

Create the IAM policy once, if it does not already exist:

``` bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

If the policy already exists, do not create a duplicate; use its
existing ARN.

### 10.2 Create the controller ServiceAccount and IAM role

The example below creates an IAM service account using eksctl. Replace
the account ID and policy ARN with your actual values:

``` bash
eksctl create iamserviceaccount \
  --cluster=k8s-2 \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --region=us-east-1 \
  --approve
```

Inspect the ServiceAccount:

``` bash
kubectl get serviceaccount aws-load-balancer-controller -n kube-system
kubectl describe serviceaccount aws-load-balancer-controller -n kube-system
```

The annotation should contain the IAM role ARN:

``` text
eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/<CONTROLLER_ROLE>
```

The IAM role trust policy must trust the cluster's OIDC provider and the
exact subject:

``` text
system:serviceaccount:kube-system:aws-load-balancer-controller
```

### 10.3 Install the Helm chart

If eksctl already created the ServiceAccount, tell Helm to reuse it. Do
not set `serviceAccount.create=true` for an existing eksctl-managed
ServiceAccount.

First, find your VPC ID:

``` bash
aws eks describe-cluster \
  --name k8s-2 \
  --region us-east-1 \
  --query 'cluster.resourcesVpcConfig.vpcId' \
  --output text
```

Install the chart, replacing `<VPC_ID>`:

``` bash
helm upgrade --install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=k8s-2 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<VPC_ID>
```

Verify:

``` bash
helm list -n kube-system
kubectl get deployment aws-load-balancer-controller -n kube-system
kubectl get pods -n kube-system
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

The controller should become Ready. If it reports `AccessDenied`, check
the IAM policy attachment and OIDC trust relationship.

------------------------------------------------------------------------

## 11. Create an ALB Ingress

The Ingress routes HTTP traffic from an AWS Application Load Balancer to
the Nginx Service.

Create `ingress.yaml`:

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: default
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 80
```

Apply and inspect:

``` bash
kubectl apply -f ingress.yaml
kubectl get ingress
kubectl describe ingress nginx-ingress
kubectl get ingress nginx-ingress -w
```

When provisioning completes, the Ingress `ADDRESS` field should show the
ALB DNS name. Open it in a browser using `http://<ALB_DNS_NAME>`.

### Ingress prerequisites

-   The AWS Load Balancer Controller must be running.
-   `nginx-service` must exist in the same namespace as the Ingress.
-   The Service port must match the backend port in the Ingress.
-   The Service selector must match the Nginx Pod labels.
-   The controller must be able to discover suitable subnets, or the
    appropriate subnet annotations must be configured.
-   Public ALB subnets need routes to an Internet Gateway and
    appropriate subnet tags.
-   Security groups and network ACLs must allow the required traffic.

For ALB Ingress, use a `ClusterIP` backend Service. Do not create a
second `LoadBalancer` Service unless you intentionally want a separate
load balancer.

------------------------------------------------------------------------

## 12. Serve your own HTML page using a ConfigMap

The default Nginx image serves `/usr/share/nginx/html/index.html`. A
ConfigMap can provide a custom `index.html` file to the Pods.

Create `index.html`:

``` html
<!DOCTYPE html>
<html>
<head>
  <title>My EKS Website</title>
  <style>
    body {
      background: #eef2ff;
      font-family: Arial, sans-serif;
      text-align: center;
      padding-top: 100px;
    }
    h1 { color: #2563eb; }
  </style>
</head>
<body>
  <h1>Welcome to My Website!</h1>
  <p>My website is running on AWS EKS.</p>
  <p>Deployed with Kubernetes Deployment, Service and ALB Ingress.</p>
</body>
</html>
```

Create or update the ConfigMap:

``` bash
kubectl create configmap nginx-html \
  --from-file=index.html \
  --dry-run=client -o yaml | kubectl apply -f -
```

Update `deployment.yaml` to mount the ConfigMap:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx-container
          image: nginx:latest
          ports:
            - containerPort: 80
          volumeMounts:
            - name: nginx-html
              mountPath: /usr/share/nginx/html/index.html
              subPath: index.html
      volumes:
        - name: nginx-html
          configMap:
            name: nginx-html
```

Apply and verify:

``` bash
kubectl apply -f deployment.yaml
kubectl rollout restart deployment nginx-deployment
kubectl rollout status deployment nginx-deployment
kubectl get pods
```

Refresh the ALB URL. You should now see your custom HTML page.

------------------------------------------------------------------------

## 13. Useful kubectl commands

``` bash
# Cluster and nodes
kubectl cluster-info
kubectl get nodes
kubectl describe node <NODE_NAME>

# Namespaces
kubectl get namespaces
kubectl get pods -A

# Deployments and ReplicaSets
kubectl get deployments
kubectl get replicasets
kubectl describe deployment nginx-deployment

# Pods
kubectl get pods
kubectl get pods -o wide
kubectl describe pod <POD_NAME>
kubectl logs <POD_NAME>

# Services and endpoints
kubectl get services
kubectl describe service nginx-service
kubectl get endpoints nginx-service

# Ingress
kubectl get ingress
kubectl describe ingress nginx-ingress

# Apply and delete resources
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
kubectl delete -f ingress.yaml
kubectl delete -f service.yaml
kubectl delete -f deployment.yaml
```

------------------------------------------------------------------------

## 14. Common errors and troubleshooting

  ----------------------------------------------------------------------------------------------
  Error                              Likely cause               What to check
  ---------------------------------- -------------------------- --------------------------------
  `i/o timeout` from kubectl         Bastion cannot reach EKS   Endpoint access, routes,
                                     API endpoint               security groups, NACLs, TCP 443

  `You must be logged in` /          IAM identity not           `aws sts get-caller-identity`,
  `Unauthorized`                     authenticated/authorized   kubeconfig, EKS access entry

  `Forbidden`                        Identity authenticated but EKS access policy / Kubernetes
                                     lacks Kubernetes           RBAC
                                     permissions                

  `Backend service does not exist`   Ingress backend Service is `kubectl get svc -A`; confirm
                                     missing or in another      name and namespace
                                     namespace                  

  Ingress ADDRESS is empty           ALB has not been           Controller Pods/logs, Ingress
                                     provisioned                Events, subnet tags, IAM role

  `DNS_PROBE_FINISHED_NXDOMAIN`      DNS name is incorrect or   Copy the exact ADDRESS from
                                     ALB is not provisioned     `kubectl get ingress`

  Nginx default page appears         Custom HTML ConfigMap not  ConfigMap, volumeMount, Pod
                                     mounted or Deployment not  rollout
                                     updated                    

  Helm says ServiceAccount cannot be ServiceAccount already     Install with
  imported                           exists and is managed by   `serviceAccount.create=false`
                                     eksctl                     

  GitHub rejects a file over 100 MB  Large binary is in a       Remove it from all pushed Git
                                     commit being pushed        history and ignore it
  ----------------------------------------------------------------------------------------------

Useful diagnostic commands:

``` bash
kubectl describe ingress nginx-ingress
kubectl get events -A --sort-by=.metadata.creationTimestamp
kubectl get pods -n kube-system
kubectl logs -n kube-system deployment/aws-load-balancer-controller
kubectl get svc -A
kubectl get endpoints nginx-service
aws sts get-caller-identity
```

------------------------------------------------------------------------

## 15. Git: save this project in GitHub

Create a `.gitignore` file:

``` gitignore
# Downloaded binaries and archives
eksctl
kubectl
helm
*.tar.gz

# Local credentials and kubeconfig
.aws/
.kube/
.env
*.pem
```

Review files before committing. Never commit AWS access keys, secret
keys, tokens, kubeconfig files containing sensitive data, or private
keys.

Configure Git identity:

``` bash
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
```

Check repository and commit files:

``` bash
git status
git branch
git remote -v
git log --oneline
git show --name-only --oneline HEAD
```

Stage, commit, and push:

``` bash
git add .
git status
git commit -m "Add EKS Kubernetes practical project"
git push -u origin main
```

GitHub HTTPS authentication uses a Personal Access Token (PAT), not your
GitHub account password. Keep the token private.

**Important large-file reminder:** Adding a binary to `.gitignore`
prevents future additions, but does not remove it from older commits. If
a file larger than GitHub's 100 MB limit is already in the history being
pushed, rewrite the unpushed history to remove it before pushing.

------------------------------------------------------------------------

## 16. Cleanup and cost awareness

AWS resources can continue generating charges even when no one is using
the application.

Delete Kubernetes resources you created when finished:

``` bash
kubectl delete -f ingress.yaml
kubectl delete -f service.yaml
kubectl delete -f deployment.yaml
kubectl delete configmap nginx-html
```

Deleting the Ingress asks the AWS Load Balancer Controller to remove the
ALB it manages. Confirm the ALB and related resources are deleted in the
EC2 Load Balancers console.

Also review AWS resources created for the lab, including EKS cluster,
worker nodes, EC2 Bastion Host, NAT Gateway, EBS volumes, Elastic IPs,
and load balancers. Delete only resources you no longer need and
understand the impact before deleting the cluster or networking.

------------------------------------------------------------------------

## 17. End-to-end verification checklist

-   [ ] `aws sts get-caller-identity` returns the intended IAM identity.
-   [ ] `kubectl get nodes` shows Ready nodes.
-   [ ] `kubectl get deployment nginx-deployment` shows 3/3 available
    replicas.
-   [ ] `kubectl get pods` shows three Ready Nginx Pods.
-   [ ] `kubectl get svc nginx-service` shows a ClusterIP Service.
-   [ ] `kubectl get endpoints nginx-service` shows Pod IP addresses.
-   [ ] AWS Load Balancer Controller Deployment is Ready.
-   [ ] `kubectl get ingress nginx-ingress` shows an ALB DNS address.
-   [ ] The ALB DNS opens the Nginx website.
-   [ ] The browser displays the custom HTML page.

## Learning summary

This project demonstrates how an application is deployed and exposed on
Amazon EKS:

1.  AWS IAM and EKS access entries control who can administer the
    cluster.
2.  OIDC and IRSA allow Pods to access AWS services using temporary
    credentials.
3.  A Deployment maintains the desired number of Nginx Pods.
4.  A ClusterIP Service provides stable internal access to those Pods.
5.  The AWS Load Balancer Controller provisions an ALB from an Ingress
    resource.
6.  The Ingress routes external HTTP traffic to the Nginx Service.
7.  A ConfigMap supplies custom HTML content to the Nginx containers.

------------------------------------------------------------------------

## 18. IAM policies and OIDC for Bastion, GitHub Actions, and the Load Balancer Controller

This section adds the IAM and OIDC setup used in the lab. There are
**three separate identities/integrations** here. Do not mix their OIDC
providers or IAM roles.

1.  **Bastion Host IAM identity**: lets an operator use AWS CLI and
    obtain EKS cluster connection details.
2.  **GitHub Actions OIDC role**: lets a GitHub Actions workflow request
    temporary AWS credentials to publish images to ECR and deploy to
    EKS.
3.  **AWS Load Balancer Controller IRSA role**: lets the controller Pod
    call AWS APIs to create and manage load balancers.

Use the same AWS account as the target EKS cluster unless you are
intentionally configuring cross-account access. Replace all placeholders
below with values from your own account and repository. The account IDs
and OIDC IDs shown in copied examples must not be reused blindly.

### 18.1 Bastion Host IAM policy

A minimal policy for retrieving EKS cluster metadata can include
`eks:DescribeCluster`. `eks:ListClusters` is useful for listing clusters
but is not required by `aws eks update-kubeconfig` when the cluster name
is already known.

Example policy document, saved as `eks-bastion-policy.json`:

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "eks:ListClusters"
      ],
      "Resource": "*"
    }
  ]
}
```

Create the policy (if it does not already exist):

``` bash
aws iam create-policy \
  --policy-name eks-bastion-policy \
  --policy-document file://eks-bastion-policy.json
```

Attach the policy to the IAM role used by the EC2 Bastion Host. The role
also needs to be authorized in EKS through an EKS access entry (or
another configured EKS authentication method). **The IAM policy above
alone does not grant Kubernetes API permissions.**

Check the identity actually used by the Bastion Host:

``` bash
aws sts get-caller-identity
```

Update kubeconfig:

``` bash
aws eks update-kubeconfig --region us-east-1 --name k8s-2
```

Install `kubectl` (the following is the historical EKS 1.30 download
used in this lab; choose a kubectl version compatible with your
cluster):

``` bash
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.30.0/2024-05-12/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version --client
```

### 18.2 EKS access entry for the Bastion IAM principal

The principal in `create-access-entry` and `associate-access-policy`
must be the same IAM principal. Do not create an access entry for a role
and then associate the policy with a different user.

Example for an EC2 role:

``` bash
aws eks create-access-entry \
  --cluster-name k8s-2 \
  --region us-east-1 \
  --principal-arn arn:aws:iam::<ACCOUNT_ID>:role/eks-bastion-role \
  --type STANDARD

aws eks associate-access-policy \
  --cluster-name k8s-2 \
  --region us-east-1 \
  --principal-arn arn:aws:iam::<ACCOUNT_ID>:role/eks-bastion-role \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```

`AmazonEKSClusterAdminPolicy` grants broad administrator access. Use it
only for a trusted administrator/lab identity. For normal operations,
assign a more restricted EKS access policy or Kubernetes RBAC
permissions.

If you choose to authorize an IAM user instead, use that same user's ARN
in both commands. The IAM identity reported by
`aws sts get-caller-identity` must be the identity you intend to
authorize.

### 18.3 GitHub Actions OIDC provider

GitHub Actions OIDC is separate from the EKS cluster OIDC issuer used by
IRSA.

In IAM, create an OpenID Connect provider with:

-   Provider URL: `https://token.actions.githubusercontent.com`
-   Audience / client ID: `sts.amazonaws.com`

You can create it in the IAM console or with the AWS CLI:

``` bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com
```

If the provider already exists, do not create a duplicate. Check:

``` bash
aws iam list-open-id-connect-providers
```

AWS manages certificate thumbprint requirements for supported OIDC
providers through its current IAM workflow. Do not copy an old
thumbprint from an unrelated example without validating it against
current AWS guidance.

### 18.4 GitHubActionRole permissions policy

This example allows a GitHub Actions workflow to authenticate to ECR and
describe EKS clusters. Save as `github-actions-policy.json` and create
an IAM policy from it.

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ECRPushPull",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DescribeEKSCluster",
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster"
      ],
      "Resource": "arn:aws:eks:us-east-1:<ACCOUNT_ID>:cluster/<EKS_CLUSTER_NAME>"
    }
  ]
}
```

`ecr:GetAuthorizationToken` requires `"Resource": "*"`. Where supported,
scope the remaining ECR actions to the specific repository ARN instead
of all resources. Add only the additional permissions your workflow
actually needs.

Create the policy:

``` bash
aws iam create-policy \
  --policy-name GitHubActionsECRAndEKSAccess \
  --policy-document file://github-actions-policy.json
```

Attach it to `GitHubActionRole`.

### 18.5 GitHubActionRole trust policy

The role trust policy allows a workflow from a specific GitHub
repository to assume the role using OIDC. Save as
`github-actions-trust-policy.json`.

Replace `<ACCOUNT_ID>`, `<GITHUB_OWNER>`, `<GITHUB_REPOSITORY>`, and the
branch name. Use the exact repository owner/name; do not use a guessed
owner string.

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:<GITHUB_OWNER>/<GITHUB_REPOSITORY>:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

This example trusts only the `main` branch of one repository. If the
workflow uses a GitHub Environment, the `sub` claim format may include
the environment name; adjust the trust policy to match the actual
workflow identity. Avoid broad patterns such as `repo:*/*` unless there
is a deliberate security reason.

Create the IAM role using this trust policy, then attach the permissions
policy:

``` bash
aws iam create-role \
  --role-name GitHubActionRole \
  --assume-role-policy-document file://github-actions-trust-policy.json
```

### 18.6 Authorize GitHubActionRole in EKS

If the GitHub Actions workflow runs `kubectl` against the cluster, its
IAM role must also be authorized by EKS. The following lab example
grants cluster-admin access; this is broad and should be replaced with
least-privilege deployment access for production.

``` bash
aws eks create-access-entry \
  --cluster-name <EKS_CLUSTER_NAME> \
  --region us-east-1 \
  --principal-arn arn:aws:iam::<ACCOUNT_ID>:role/GitHubActionRole \
  --type STANDARD

aws eks associate-access-policy \
  --cluster-name <EKS_CLUSTER_NAME> \
  --region us-east-1 \
  --principal-arn arn:aws:iam::<ACCOUNT_ID>:role/GitHubActionRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```

Do not run these commands against a different cluster/account from the
one intended. If the access entry already exists, do not create it
again; inspect the existing entry and its associated policies.

### 18.7 EKS cluster OIDC issuer for the Load Balancer Controller

The EKS cluster has its own OIDC issuer. This is **not**
`token.actions.githubusercontent.com`.

Get the issuer URL:

``` bash
aws eks describe-cluster \
  --name k8s-2 \
  --region us-east-1 \
  --query 'cluster.identity.oidc.issuer' \
  --output text
```

For IRSA, the recommended `eksctl` method is:

``` bash
eksctl utils associate-iam-oidc-provider \
  --cluster k8s-2 \
  --region us-east-1 \
  --approve
```

This associates the EKS OIDC provider with the AWS account. Avoid
manually copying a cluster OIDC URL, OIDC ID, AWS account ID, or
certificate thumbprint from another cluster or account.

### 18.8 AWS Load Balancer Controller IAM policy and role

Download the IAM policy matching the controller version you install.
Example:

``` bash
curl -o iam_policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.1/docs/install/iam_policy.json
```

Create the IAM policy once, if it does not already exist, and attach it
to the controller IAM role. The policy name used in this lab is
`AWSLoadBalancerControllerIAMPolicy`.

The role trust policy must trust the **EKS cluster OIDC provider** and
the controller ServiceAccount subject:

``` text
system:serviceaccount:kube-system:aws-load-balancer-controller
```

Conceptual trust-policy structure (replace the OIDC provider path and
account ID with the values from your own cluster):

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/<EKS_OIDC_ID>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/<EKS_OIDC_ID>:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller",
          "oidc.eks.us-east-1.amazonaws.com/id/<EKS_OIDC_ID>:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

Create the ServiceAccount with `eksctl` (example; use your real policy
ARN):

``` bash
eksctl create iamserviceaccount \
  --cluster=k8s-2 \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --region=us-east-1 \
  --approve
```

Verify the ServiceAccount and role annotation:

``` bash
kubectl get serviceaccount aws-load-balancer-controller -n kube-system
kubectl describe serviceaccount aws-load-balancer-controller -n kube-system
```

Add the AWS Helm repository and update it:

``` bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

Install or upgrade the controller. Since `eksctl` created the
ServiceAccount, set `serviceAccount.create=false` so Helm reuses it:

``` bash
helm upgrade --install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=k8s-2 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<VPC_ID>
```

Verify:

``` bash
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl get pods -n kube-system
helm list -A
```

### 18.9 Subnet tags required for ALB discovery

The AWS Load Balancer Controller needs suitable subnets to place an ALB.
Ensure the subnet tags are correct:

  Subnet purpose        Required tag                        Value
  --------------------- ----------------------------------- -------
  Public ALB subnet     `kubernetes.io/role/elb`            `1`
  Internal ALB subnet   `kubernetes.io/role/internal-elb`   `1`

Public subnets also need a route to an Internet Gateway for an
internet-facing ALB. Internal subnets are used for internal load
balancers and should not be made public merely to satisfy controller
discovery.

Check the subnet IDs and tags in the VPC console or with AWS CLI. Also
ensure the cluster ownership/shared tags required by the controller and
your EKS version/configuration are present.

------------------------------------------------------------------------

## 19. Important identity and security distinctions

  ---------------------------------------------------------------------------------------
  Identity / provider                     Used by                 Purpose
  --------------------------------------- ----------------------- -----------------------
  Bastion IAM role                        EC2 Bastion Host        AWS CLI access and EKS
                                                                  cluster authentication

  EKS access entry                        IAM principal           Authorizes the IAM
                                                                  principal to Kubernetes

  GitHub OIDC provider                    GitHub Actions          Allows workflows to
                                                                  assume an AWS IAM role
                                                                  without stored AWS keys

  `GitHubActionRole`                      GitHub Actions workflow ECR operations and, if
                                                                  authorized, EKS
                                                                  deployments

  EKS cluster OIDC provider               Kubernetes              Enables IRSA for Pods
                                          ServiceAccount          

  `AmazonEKSLoadBalancerControllerRole`   Load Balancer           Calls AWS APIs to
                                          Controller Pod          provision and manage
                                                                  load balancers
  ---------------------------------------------------------------------------------------

**Common mistakes to avoid:**

-   Do not use the GitHub OIDC provider URL as the EKS cluster OIDC
    provider.
-   Do not copy an OIDC ID, thumbprint, account ID, role ARN, or
    repository name from another environment.
-   The IAM principal in an EKS access entry must be the same principal
    to which you associate the access policy.
-   `eks:DescribeCluster` permission alone does not grant Kubernetes
    administrator permissions.
-   `serviceAccount.create=false` is appropriate when Helm should reuse
    an existing ServiceAccount created by `eksctl`.
-   Do not grant cluster-admin to a CI role unless absolutely necessary;
    use least-privilege deployment permissions.
-   Never commit IAM access keys, GitHub tokens, kubeconfig credentials,
    or private keys.

