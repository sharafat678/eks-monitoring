# Prometheus and grafana setup on EKS

This guide walks you through deploying the **Prometheus** and **Grafana** on an Amazon EKS cluster using `AWS CLI`, `kubectl`, `eksctl`, and `Helm`. The process includes creating the cluster, setting up the necessary IAM roles and policies, and deploying **Prometheus** and **grafana**

Pod Identity association and IAM Roles for Service Accounts (IRSA) are two approaches to providing fine-grained access control for applications running in Amazon EKS (Elastic Kubernetes Service). Both methods allow your applications to access AWS resources securely without the need to manage static credentials. But in some organizations it is not allowed to create OIDC provider. So, in this tutorial instead of IRSA we use Pod identity association to access aws resources.

## Prerequisites

1. **Tools:** Ensure you have the following installed and configured:
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html#getting-started-install-instructions)
  
```bash
   curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
   unzip awscliv2.zip
   sudo ./aws/install
   aws --version

```
   - [kubectl](https://kubernetes.io/docs/tasks/tools/)
  ```bash
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.31.3/2024-12-12/bin/linux/amd64/kubectl

chmod +x ./kubectl

./kubectl version

sudo mv ./kubectl /usr/local/bin/     

kubectl version --client
```
- [eksctl](https://eksctl.io/)

```bash
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctinux_amd64.tar.gz"
   
tar -xzf eksctinux_amd64.tar.gz

Chmod +x eksctl

sudo mv eksctl /usr/local/bin

eksctl version
```


- [Helm](https://helm.sh/docs/intro/install/)
  
```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null

sudo apt-get install apt-transport-https --yes

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/
helm-stable-debian.list

sudo apt-get update

sudo apt-get install helm
```


**Important NOTE：** Your AWS-CLI user must have sufficient permissions to create and manage EKS clusters. Example permissions include:
   - `eks:*`
   - `ec2:*`
   - `iam:*`
   - `elasticloadbalancing:*`


## Steps:

### 1. Create an EKS Cluster and Nodegroup

1.1 Create a cluster-config.yml file 

```bash
cat > cluster-config.yml <<EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-eks1
  region: cn-northwest-1
  version: "1.31"

vpc:
  subnets:
    private:
      cn-northwest-1a:
        id: subnet-055e42c6aabfa #enter your subnet-id here
      cn-northwest-1b:
        id: subnet-0e215fcaed0a #enter your subnet-id here

addons:
  - name: coredns
    attachPolicyARNs:
      - arn:aws-cn:iam::aws:policy/AmazonEKS_CNI_Policy
    configurationValues: "{}"

  - name: eks-pod-identity-agent
    configurationValues: "{}"

  - name: kube-proxy
    configurationValues: "{}"

  - name: vpc-cni
    attachPolicyARNs:
      - arn:aws-cn:iam::aws:policy/AmazonEKS_CNI_Policy
    configurationValues: "{}"

  - name: aws-ebs-csi-driver
    configurationValues: "{}"
EOF
```
Run the following command to create an EKS cluster:

```bash
eksctl create cluster -f cluster-config.yml
```


1.2 Create a nodegroup.yml file 

```bash
cat > nodegroup.yaml <<EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-eks1
  region: cn-northwest-1

managedNodeGroups:
  - name: node2
    instanceType: t2.medium
    desiredCapacity: 2
    minSize: 1
    maxSize: 3
    privateNetworking: true
    subnets:
      - subnet-2
      - subnet-2
    iam:
      attachPolicyARNs:
        - arn:aws-cn:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws-cn:iam::aws:policy/AmazonEKS_CNI_Policy
        - arn:aws-cn:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

      withAddonPolicies:
        ebs: true
EOF
```
Run the following command to create nodegroup:

```bash
eksctl create nodegroup -f nodegroup.yml
```

### 2. Setup monitoring using Helm carts

Create monitoring namespace

```bash
kubectl create namespace monitoring
```
Following command will install Prometheus, Alertmanager, and Grafana. Persistent Volume Claims (PVCs) are needed to ensure that Prometheus, Alertmanager, and Grafana retain data even if the pods are restarted or rescheduled.

```bash
wget https://github.com/sharafat678/eks-monitoring/raw/main/kube-prometheus-stack.zip -O prometheus-stack.zip

unzip prometheus-stack.zip -d prometheus-stack

helm upgrade --install prometheus ./prometheus-stack/kube-prometheus-stack -n monitoring \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=gp2 \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.accessModes[0]=ReadWriteOnce \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=10Gi \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.storageClassName=gp2 \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.accessModes[0]=ReadWriteOnce \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.resources.requests.storage=10Gi \
  --set grafana.persistence.enabled=true \
  --set grafana.persistence.storageClassName=gp2 \
  --set grafana.persistence.size=10Gi \
  --wait

```
**Important Note:** We can directly use the helm repo add command to add it as a Helm repository. However, in some countries with restrictions, like China, access to GitHub may be blocked. The easiest workaround is to first download the chart to your current directory and then install it using Helm.


```bash
kubectl get po -n monitoring
```
- **Description:** You will notice that after this monitoring pods are created and it should be in running state.

---

### 3. Create IAM Policy and role for ALB Controller

Later on when we want to expose our monitoring services prometheus and grafana we need and alb which will be created through ingress, and our controller pod must have permissions to create resources in aws. So, create `alb-controller-policy.json` file with the following content:

```bash
cat > alb-controller-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "iam:CreateServiceLinkedRole"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "iam:AWSServiceName": "elasticloadbalancing.amazonaws.com"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeAccountAttributes",
                "ec2:DescribeAddresses",
                "ec2:DescribeAvailabilityZones",
                "ec2:DescribeInternetGateways",
                "ec2:DescribeVpcs",
                "ec2:DescribeVpcPeeringConnections",
                "ec2:DescribeSubnets",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeInstances",
                "ec2:DescribeNetworkInterfaces",
                "ec2:DescribeTags",
                "ec2:GetCoipPoolUsage",
                "ec2:DescribeCoipPools",
                "elasticloadbalancing:DescribeLoadBalancers",
                "elasticloadbalancing:DescribeLoadBalancerAttributes",
                "elasticloadbalancing:DescribeListeners",
                "elasticloadbalancing:DescribeListenerCertificates",
                "elasticloadbalancing:DescribeSSLPolicies",
                "elasticloadbalancing:DescribeRules",
                "elasticloadbalancing:DescribeTargetGroups",
                "elasticloadbalancing:DescribeTargetGroupAttributes",
                "elasticloadbalancing:DescribeTargetHealth",
                "elasticloadbalancing:DescribeTags"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "cognito-idp:DescribeUserPoolClient",
                "acm:ListCertificates",
                "acm:DescribeCertificate",
                "iam:ListServerCertificates",
                "iam:GetServerCertificate",
                "waf-regional:GetWebACL",
                "waf-regional:GetWebACLForResource",
                "waf-regional:AssociateWebACL",
                "waf-regional:DisassociateWebACL",
                "wafv2:GetWebACL",
                "wafv2:GetWebACLForResource",
                "wafv2:AssociateWebACL",
                "wafv2:DisassociateWebACL",
                "shield:GetSubscriptionState",
                "shield:DescribeProtection",
                "shield:CreateProtection",
                "shield:DeleteProtection"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:AuthorizeSecurityGroupIngress",
                "ec2:RevokeSecurityGroupIngress"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:CreateSecurityGroup"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:CreateTags"
            ],
            "Resource": "arn:aws-cn:ec2:*:*:security-group/*",
            "Condition": {
                "StringEquals": {
                    "ec2:CreateAction": "CreateSecurityGroup"
                },
                "Null": {
                    "aws:RequestTag/elbv2.k8s.aws/cluster": "false"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:CreateTags",
                "ec2:DeleteTags"
            ],
            "Resource": "arn:aws-cn:ec2:*:*:security-group/*",
            "Condition": {
                "Null": {
                    "aws:RequestTag/elbv2.k8s.aws/cluster": "true",
                    "aws:ResourceTag/elbv2.k8s.aws/cluster": "false"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:AuthorizeSecurityGroupIngress",
                "ec2:RevokeSecurityGroupIngress",
                "ec2:DeleteSecurityGroup"
            ],
            "Resource": "*",
            "Condition": {
                "Null": {
                    "aws:ResourceTag/elbv2.k8s.aws/cluster": "false"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "elasticloadbalancing:CreateLoadBalancer",
                "elasticloadbalancing:CreateTargetGroup"
            ],
            "Resource": "*",
            "Condition": {
                "Null": {
                    "aws:RequestTag/elbv2.k8s.aws/cluster": "false"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "elasticloadbalancing:CreateListener",
                "elasticloadbalancing:DeleteListener",
                "elasticloadbalancing:CreateRule",
                "elasticloadbalancing:DeleteRule"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "elasticloadbalancing:AddTags",
                "elasticloadbalancing:RemoveTags"
            ],
            "Resource": [
                "arn:aws-cn:elasticloadbalancing:*:*:targetgroup/*/*",
                "arn:aws-cn:elasticloadbalancing:*:*:loadbalancer/net/*/*",
                "arn:aws-cn:elasticloadbalancing:*:*:loadbalancer/app/*/*"
            ],
            "Condition": {
                "Null": {
                    "aws:RequestTag/elbv2.k8s.aws/cluster": "true",
                    "aws:ResourceTag/elbv2.k8s.aws/cluster": "false"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "elasticloadbalancing:AddTags",
                "elasticloadbalancing:RemoveTags"
            ],
            "Resource": [
                "arn:aws-cn:elasticloadbalancing:*:*:listener/net/*/*/*",
                "arn:aws-cn:elasticloadbalancing:*:*:listener/app/*/*/*",
                "arn:aws-cn:elasticloadbalancing:*:*:listener-rule/net/*/*/*",
                "arn:aws-cn:elasticloadbalancing:*:*:listener-rule/app/*/*/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "elasticloadbalancing:ModifyLoadBalancerAttributes",
                "elasticloadbalancing:SetIpAddressType",
                "elasticloadbalancing:SetSecurityGroups",
                "elasticloadbalancing:SetSubnets",
                "elasticloadbalancing:DeleteLoadBalancer",
                "elasticloadbalancing:ModifyTargetGroup",
                "elasticloadbalancing:ModifyTargetGroupAttributes",
                "elasticloadbalancing:DeleteTargetGroup"
            ],
            "Resource": "*",
            "Condition": {
                "Null": {
                    "aws:ResourceTag/elbv2.k8s.aws/cluster": "false"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "elasticloadbalancing:AddTags"
            ],
            "Resource": [
                "arn:aws-cn:elasticloadbalancing:*:*:targetgroup/*/*",
                "arn:aws-cn:elasticloadbalancing:*:*:loadbalancer/net/*/*",
                "arn:aws-cn:elasticloadbalancing:*:*:loadbalancer/app/*/*"
            ],
            "Condition": {
                "StringEquals": {
                    "elasticloadbalancing:CreateAction": [
                        "CreateTargetGroup",
                        "CreateLoadBalancer"
                    ]
                },
                "Null": {
                    "aws:RequestTag/elbv2.k8s.aws/cluster": "false"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "elasticloadbalancing:RegisterTargets",
                "elasticloadbalancing:DeregisterTargets"
            ],
            "Resource": "arn:aws-cn:elasticloadbalancing:*:*:targetgroup/*/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "elasticloadbalancing:SetWebAcl",
                "elasticloadbalancing:ModifyListener",
                "elasticloadbalancing:AddListenerCertificates",
                "elasticloadbalancing:RemoveListenerCertificates",
                "elasticloadbalancing:ModifyRule"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "elasticloadbalancing:DescribeListenerAttributes",
            "Resource": "*"
        }
    ]
}
EOF
```

Create the policy in AWS:

```bash
aws iam create-policy \
  --policy-name alb-Policy \
  --policy-document file://alb-controller-policy.json
```


Create `trust-policy.json` file with following content:

```bash
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

Create IAM role for ALB with this trust policy in AWS:

```bash
aws iam create-role \
  --role-name alb-controller-role \
  --assume-role-policy-document file://trust-policy.json \
  --region cn-northwest-1
```

Attach IAM policy with the role:

```bash
aws iam attach-role-policy \
  --role-name alb-controller-role \
  --policy-arn arn:aws-cn: \  #ARN of the policy you created above named as alb-Policy
  --region cn-northwest-1

```

- **Description:** This role allows the ALB controller to manage AWS resources required for ingress.
---

### 4. Pod Identity Association
EKS Pod Identity is a method to associate pods with AWS IAM roles and service accounts for the workload to use. The following command associates the IAM role (alb-controller-role) with the Kubernetes service account (monitoring-sa).
When a pod runs with this service account, it can assume the IAM role and access AWS resources based on its permissions.

Create service account in monitoring namespace
```bash
kubectl create serviceaccount monitoring-sa -n monitoring
```

```bash
aws eks create-pod-identity-association \
  --cluster-name my-eks1 \
  --namespace monitoring \
  --service-account monitoring-sa \
  --role-arn arn:aws-cn:iam::<Account-id>:role/alb-controller-role \
  --region cn-northwest-1

```

### 5. Install ALB Controller using Helm

```bash
wget https://github.com/sharafat678/eks-monitoring/raw/main/aws-load-balancer-controller.zip -O alb-controller.zip

unzip alb-controller.zip -d alb-controller

helm install aws-load-balancer-controller ./alb-controller/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-eks1 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=monitoring-sa \
  --set region=cn-northwest-1 \
  --set vpcId=vpc-id    #replace vpc id here
```
- **Description:** The Helm command download the helm charts for alb-controller, unzip the charts and sets up the AWS Load Balancer Controller(pod). The controller will create ALBs dynamically based on the Kubernetes resources (like Ingress or Service) you deploy. This controller is necessary if you want to access your application outside of the vpc.
---

### 6. Create the ingress resource

```bash
cat > ingress.yml <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: monitoring
  name: monitoring-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: "internet-facing"  # Make it public
    alb.ingress.kubernetes.io/target-type: "ip"  # Route directly to service IPs #alb.ingress.
    alb.ingress.kubernetes.io/success-codes: 200,302
spec:
  ingressClassName: alb  # Still using AWS Load Balancer Controller
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-grafana
                port:
                  number: 80
          - path: /prometheus
            pathType: Prefix
            backend:
              service:
                name: prometheus-kube-prometheus-prometheus
                port:
                  number: 9090  # Requests on port 9090 go to Prometheus

          - path: /alertmanager
            pathType: Prefix
            backend:
              service:
                name: prometheus-kube-prometheus-alertmanager  
                port:
                  number: 9093  # Requests on port 9090 go to Prometheus
EOF
```

Run the command to create ingress resource.
```bash
kubectl apply -f ingress.yml
```
To see the ingress resource
```bash
kubectl get ingress -n monitoring
```

- You can see the address of the ALB. Simply paste that address into your browser to access the Grafana web page. To access Alertmanager and Prometheus, append /alertmanager and /prometheus to the ALB DNS.

---


## Cleanup
To delete the EKS cluster and associated resources:

```bash
eksctl delete cluster --name my-eks1
```