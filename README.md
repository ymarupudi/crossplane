# crossplane

This repository is for storing crossplane composite resource definitions, compositions and claim files.

It contains folder for each resource type and its files within each of the resource type folder.
# Prerequisites

You need an IAM access key pair, AWS CLI, bash, kubectl, helm and crossplane in your machine.
# For AWS CLI
Run below command in PowerShell to install AWS CLI. Click [here](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) for the latest AWS CLI release
```console
https://awscli.amazonaws.com/AWSCLIV2.msi
```

Run below command to set up AWS CLI installation
```console
aws configure
```
<b>AWS Access Key ID [None]:</b> ENTER-YOUR-ACCESS-KEY-ID

<b>AWS Secret Access Key [None]:</b> ENTER-YOUR-SECRET-ACCESS-KEY

<b>Default region name [None]:</b> ENTER-REGION (example: us-east-1)

<b>Default output format [None]:</b> json
# For GitBash
Click [here](https://github.com/git-for-windows/git/releases/download/v2.36.1.windows.1/Git-2.36.1-64-bit.exe) to download GitBash and then install it. Click [here](https://git-scm.com/download/win) for the latest GitBash release
# For kubectl
Click [here](https://s3.us-west-2.amazonaws.com/amazon-eks/1.22.6/2022-03-09/bin/windows/amd64/kubectl.exe) to download kubectl. Click [here](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html) for the latest kubectl release
# For helm
Click [here](https://get.helm.sh/helm-v3.9.0-rc.1-windows-amd64.zip) to download helm. Click [here](https://github.com/helm/helm/releases) for the latest helm release
# For eksctl
Click [here](https://github.com/weaveworks/eksctl/releases/download/v0.116.0/eksctl_Windows_amd64.zip) to download eksctl. Click [here](https://github.com/weaveworks/eksctl/releases) for the latest eksctl release
# For crossplane
Run below command in bash
```console
curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh
```
Run below command to move the crossplane file to the folder where kubectl application is present
```console
mv kubectl-crossplane $(dirname $(which kubectl))/kubectl-crossplane.exe
```
Extract helm and eksctl zip files.

Move kubectl, helm and eksctl applications to the folder of your choice and add that folder to the system file path to make it easily accessible from command line.
# Follow below steps to install Crossplane on Kubernetes Cluster
Run below command to list the EKS clusters in your AWS account in the specified region
```console
aws eks list-clusters --profile default --region us-east-1
```
Run below command to configure kubectl so that you can connect to the specified EKS cluster
```console
aws eks update-kubeconfig --profile default --name=ENTER-YOUR-EKS-CLUSTER-NAME --region us-east-1
```
Add crossplane-stable repository to helm using below command
```console
helm repo add crossplane-stable https://charts.crossplane.io/stable
```
Run below command to update helm repositories
```console
helm repo update
```
Run below command to install crossplane on Kubernetes Cluster
```console
helm install crossplane --create-namespace --namespace crossplane-system crossplane-stable/crossplane
```
Use below command to check whether crossplane components are up and healthy
```console
kubectl get all -n crossplane-system
````
Use below command in bash to generate configuration file with the configured AWS Credentials in your machine
```console
AWS_PROFILE=default && echo -e "[default]\naws_access_key_id = $(aws configure get aws_access_key_id --profile $AWS_PROFILE)\naws_secret_access_key = $(aws configure get aws_secret_access_key --profile $AWS_PROFILE)" > creds.conf
```
Use below command in bash to create a Kubernetes secret by using the configuration file
```console
kubectl create secret generic aws-creds -n crossplane-system --from-file=creds=./creds.conf
```
Use below command to check kubernetes secret
```console
kubectl get secrets -n crossplane-system
```
Run below lines to create <b>provider-aws.yaml</b> file
```console
cat > provider-aws.yaml <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: crossplane/provider-aws:v0.32.0
EOF
```
Run below command to install <b>provider-aws</b>
```console
kubectl apply -f provider-aws.yaml
```
Run below lines to create <b>provider-aws-jet.yaml</b> file
```console
cat > provider-aws-jet.yaml <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-jet
spec:
  package: crossplane/provider-jet-aws:v0.5.0-preview
EOF
```
Run below command to install <b>provider-aws-jet</b>
```console
kubectl apply -f provider-aws-jet.yaml
```
Wait for the provider to be healthy, check provider status using below command
```console
kubectl get providers
```
Run below lines to create <b>providerconfig.yaml</b> file
```console
cat > providerconfig.yaml <<EOF
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: provider-aws
  namespace: crossplane-system
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-creds
      key: creds
---
apiVersion: aws.jet.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: provider-aws-jet
  namespace: crossplane-system
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-creds
      key: creds
EOF
```
Run below command to create aws and jet-aws <b>providerconfig</b>
```console
kubectl apply -f providerconfig.yaml
```
Upon successful creation of providerconfig, your EKS cluster should now be able to provision infrastructure in AWS Cloud.
# Method 2
### Authenticating to AWS API using IAM role for provider ServiceAccount
Create OIDC provider for EKS cluster
```console
eksctl utils associate-iam-oidc-provider --profile <your-aws-profile-name> --cluster <your-eks-cluster-name> --region <your-region> --approve
```
Create trust relationship for IAM role
```console
cat > trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "<your-identity-provider-arn>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringLike": {
          "<your-identity-provider-name>:sub": "system:serviceaccount:crossplane-system:provider-aws-*"
        }
      }
    }
  ]
}
EOF
```
#### 7. Create an IAM role with above trust relationship
```console
aws iam create-role --profile <your-aws-profile-name> --role-name <enter-role-name> --assume-role-policy-document file://trust.json
```
#### 8. Associate a policy with the IAM role. Should select a policy with permissions to `Services` for provisioning resources.
```console
aws iam attach-role-policy --profile <your-aws-profile-name> --role-name <enter-role-name> --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```
#### 9. Create `ControllerConfig` and `Provider`
```console
cat > provider.yaml <<EOF
apiVersion: pkg.crossplane.io/v1alpha1
kind: ControllerConfig
metadata:
  name: controllerconfig-aws
  annotations:
    eks.amazonaws.com/role-arn: <enter-your-iam-role-arn-created-in-previous-step>
spec:
  podSecurityContext:
    fsGroup: 2000
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: crossplane/provider-aws:v0.28.1
  controllerConfigRef:
    name: controllerconfig-aws
---
apiVersion: pkg.crossplane.io/v1alpha1
kind: ControllerConfig
metadata:
  name: controllerconfig-aws-jet
  annotations:
    eks.amazonaws.com/role-arn: <enter-your-iam-role-arn-created-in-previous-step>
spec:
  podSecurityContext:
    fsGroup: 2000
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-jet
spec:
  package: crossplane/provider-jet-aws:v0.4.2
  controllerConfigRef:
    name: controllerconfig-aws-jet
EOF
```
#### 10. Apply `provider.yaml`
```console
kubectl apply -f provider.yaml
```
#### 11. Wait for the `Providers` to be healthy by executing
```console
kubectl get providers
```
#### 12. Create `ProviderConfig`
```console
cat > providerconfig.yaml <<EOF
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: provider-aws
spec:
  credentials:
    source: InjectedIdentity
---
apiVersion: aws.jet.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: provider-aws-jet
spec:
  credentials:
    source: InjectedIdentity
EOF
```
#### 13. Once `Providers` are healthy, apply `providerconfig.yaml`
```console
kubectl apply -f providerconfig.yaml
```
#### 14. Ensure that `ProviderConfig` was created
```console
kubectl get providerconfig.aws
kubectl get providerconfig.aws.jet
```
