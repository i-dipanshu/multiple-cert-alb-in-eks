# Deploy Django todo app in a EKS cluster and configure a ALB Ingress Controller 

#### Resources

1. **[Create](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)**

2. **[Deploy](https://docs.aws.amazon.com/eks/latest/userguide/sample-deployment.html)**


## Prerequisite 

Install the below tools in your to local machine to interact with EKS 

1. Install [awscli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) and [configure](https://docs.aws.amazon.com/cli/latest/userguide/cli-authentication-user.html) IAM access key and access id
2. Install [eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html) 
3. Install [kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)

## 1. Create the EKS resources

Now once we have all three required tools configured and ready to go, we can start creating and configuring our eks cluster. 

Pricing - [EKS](https://aws.amazon.com/eks/pricing/)

#### 1. Create the eks cluster using eksctl (Managed Linux Nodes) - [docs](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html)

```sh
eksctl create cluster --name my-cluster --region region-code
```

When first installing kubectl, it isn't yet configured to communicate with any server. We will cover this configuration as needed in other procedures. If you ever need to update the configuration to communicate with a particular cluster, you can run the following command. Replace region-code with the AWS Region that your cluster is in. Replace my-cluster with the name of your cluster.

```sh 
aws eks update-kubeconfig --region region-code --name my-cluster
```

#### 2. Create a namespace for our app
```sh
kubectl create namespace django-todo-app
```

## 2. Configure OIDC Connector
[Docs](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)

```sh
cluster_name=my-cluster
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
echo $oidc_id

# Check for existing OIDC connector
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4

# Create a new OIDC Connector, if there is no existing OIDC Connector 
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

## 3. Create a ALB Add on 
- [Docs - AWS ](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html)

- [Docs - Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.6/deploy/installation/)

- Application Load Balancing on Eks - [refer](https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html)

#### Download IAM policy

```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.6.0/docs/install/iam_policy.json
```

#### Create IAM Policy

```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

#### Create IAM Role

```
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

#### Deploy ALB controller

Add helm repo

```
helm repo add eks https://aws.github.io/eks-charts
```

Update the repo

```
helm repo update eks
```

Install

```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region> \
  --set vpcId=<your-vpc-id>
```

Verify that the deployments are running.

```
kubectl get deployment -n kube-system aws-load-balancer-controller
```

Verify that an alb is created by the controller

```sh
aws elbv2 describe-load-balancers
```

## 4. Configure Multiple cert on ALB ingress controller eks 

#### Resources
- Feature introduced in [PR 818](https://github.com/kubernetes-sigs/aws-load-balancer-controller/pull/818)
- Docs for multiple cert as arn - [refer](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.6/guide/ingress/annotations/#certificate-arn)
- Cert discovery if above arn is not described - [refer](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.6/guide/ingress/cert_discovery/)


## 5. Configure Ingress controller to use same alb for multiple ingress resources
[Docs](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.6/guide/ingress/annotations/#ingressgroup)


## Problems 
1. Ingress controller is created but it doesn't create an alb 
    possible fix - [refer](https://repost.aws/questions/QUGNQwcRe4SU6BjhZYpHixXg/eks-aws-load-balancer-controller-ingress-created-but-the-alb-is-not)
