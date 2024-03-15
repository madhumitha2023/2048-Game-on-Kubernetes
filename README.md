2048-Game-on-Kubernetes

🚀 Deploying Real-Time 2048 Game on Kubernetes with EKS and Fargate

In this project, we'll deploy a real-time 2048 game application on Amazon EKS (Elastic Kubernetes Service) using AWS Fargate for compute and an Application Load Balancer (ALB) for routing traffic to our pods through service.

Prerequisites:
Install eksctl, kubectl, aws cli, and configure aws cli with access key and secret key.

Steps involved:
pre-requisite: Install eksctl, kubectl, aws cli and configure aws cli using access key and secret key
Use eksctl a very good utility to create a entire cluster -> "eksctl create cluster --name my-cluster --region region-code --fargate"
This eksctl creates an entire configuration for us, It also creates a vpc, public subnet and private subnet. In Private subnet I will place my application.
Download the kubeconfig file (yaml file with all the kubernetes cluster details, certificates and secret tokens to authenticate the cluster) -> "aws eks update-kubeconfig --name cluster-name --region region-code"
Deployment of actual application:
Create a fargateprofile-> "eksctl create fargateprofile --cluster cluster-name --region region-code --name alb-sample-app --namespace game-2048"
This creates a new fargate profile with the name "alb-sample-app" for a new namespace with "game-2048"
Deploy 2048 applications pod using the deployment-> kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml"
The above file has all the configurations related to deployment, service and ingress of the application game 2048. It has the namespace "game-2048" and that is the reason we created one in our fargateprofile. This file is taken from the eks official document.  Here replica is set to 5 to manage the load. we need to consider the service for two things one is that verify the target port of service is same as the container port of the pod in deployment. Second, make sure to have proper labels and selectors. The selector in service should match with the template labels of the deployment spec data. This way my service will be able to discover the pods. Ingress is used to route the traffic inside the cluster and has the ingressClassName as "alb". Ingress application load balancer controller will read this ingress resource and whenever it finds the matching rules, it will forward the traffic to backend service 2048. Service in turn forwards the traffic to the pods available in the namespace "game-2048". We all know without ingress controller ingress resource will be useless so now lets create an Ingress Controller which not only creates the load balancer for us but also configure entire load balancer. To create ALB controller (ingress controller), the pre-requisite is to configure IAM OIDC identity provider. The reason we need this IAM OIDC connector is that, the ALB controller(kubernetes pod) running needs to access the Application load balancer. The Kubernetes pod that is ALB controller if it has to communicate with AWS resources in our case ALB, controller has to be integrated with IAM -> "eksctl utils associate-iam-oidc-provider --cluster cluster-name --region region-code --approve"
Next, for an ALB controller which is a pod grant access  to AWS services such as ALB. So creating both IAM policy and IAM role.
Download IAM policy -> "curl -o https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json"
Create IAM policy -> "aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json "
This json file is available in the ALB controller document. 
Next create service account and attach the role to the service account of the pod -> "eksctl create iamserviceaccount --cluster=<your-cluster-name> --namespace=kube-system --name=aws-load-balancer-controller --role-name AmazonEKSLoadBalancerControllerRole --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy AWSLoadBalancerControllerIAMPolicy --approve"
Now lets create the application load balancer controller using helm charts. This helm chart will create the actual controller and use the above service account created for running the pods. 
Add Helm repo -> "helm repo add eks https://aws.github.io/eks-charts"
Update the repo -> "helm repo update eks"
Install -> "helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=<your-cluster-name> --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller --set region=<region> --set vpcId=<your-vpc-id>"
Verify that the deployments are running and atleast 2 replicas are set -> "kubectl get deployment -n kube-system aws-load-balancer-controller"
To debug for errors -> Type "kubectl edit deploy/aws-load-balancer-controller -n kube-system" and navigate to the "status" field or we can also use describe command.
Now the AWS Application Load Balancer will be created. Using the DNS name of the AWS ALB(address of the ALB ingress controller) access the Game 2048 application in your favorite browser.
