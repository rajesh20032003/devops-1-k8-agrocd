🚀 Enterprise GitOps Microservices Architecture on AWS EKS
📌 Project Overview
This repository contains the declarative infrastructure and application manifests for a highly available, 4-tier Node.js microservice architecture (Frontend, API Gateway, User Service, Order Service).

The platform is deployed to an AWS Elastic Kubernetes Service (EKS) cluster using Terraform for underlying infrastructure provisioning and ArgoCD for pure GitOps continuous deployment. The architecture features stateful PostgreSQL databases, native AWS Application Load Balancing, zero-trust network policies, and a complete observability stack.

🏗️ Technical Stack
Cloud Provider: AWS (EKS, EC2, VPC, EBS, ALB, IAM)

Infrastructure as Code (IaC): Terraform

Container Orchestration: Kubernetes (v1.30)

GitOps & CI/CD: ArgoCD, Helm, Jenkins

Networking & Security: Calico (Network Policies), AWS VPC CNI, AWS Load Balancer Controller

Storage: Amazon EBS CSI Driver (gp3 class)

Observability: Prometheus, Grafana, Loki, Alertmanager

📸 Architecture & Dashboards
1. The Microservices Dashboard (Native AWS ALB)
Traffic successfully routed through the AWS Application Load Balancer to the frontend pods.
<img width="1918" height="972" alt="Screenshot 2026-05-11 182226" src="https://github.com/user-attachments/assets/91486230-6ed6-4570-8101-26a3a42be718" />


2. GitOps Continuous Deployment (ArgoCD)
Declarative state management syncing the observability stack and microservices.
<img width="1918" height="872" alt="Screenshot 2026-05-11 173404" src="https://github.com/user-attachments/assets/3304183d-91cb-4134-a310-90fd3df107e2" />



4. AWS Infrastructure Footprint
VPC Resource Map detailing the Public/Private Subnet isolation, NAT Gateways, and Route Tables.
 <img width="1918" height="833" alt="Screenshot 2026-05-11 184753" src="https://github.com/user-attachments/assets/f55708ab-47fe-44fc-91bb-d9cd66412034" />


EKS Cluster & Stateful Add-ons running on t3.xlarge nodes.
<img width="1918" height="855" alt="Screenshot 2026-05-11 172846" src="https://github.com/user-attachments/assets/7ec52e3c-c30b-401e-8495-2297fa06f138" />


🛠️ Challenges Overcome (The "War Room")
Building production-grade infrastructure is rarely seamless. Here are the core architectural challenges encountered during deployment and how they were resolved:

1. The "Minikube Hangover" (Storage Provisioning Failure)
Symptom: PostgreSQL database pods were stuck in a Pending state with "unbound immediate PersistentVolumeClaims" events.

Root Cause: The GitOps repository contained legacy StorageClass configurations using k8s.io/minikube-hostpath, which the AWS EKS cluster could not fulfill.

Resolution: Rewrote the StorageClass YAML to utilize the ebs.csi.aws.com provisioner, requesting gp3 block storage with WaitForFirstConsumer volume binding. Re-synced via ArgoCD, allowing EKS to dynamically attach physical SSDs to the worker nodes.

2. The AWS EBS ext4 Bug (Database CrashLoopBackOff)
Symptom: Once storage was bound, PostgreSQL pods immediately entered CrashLoopBackOff with Exit Code 1. Logs indicated: directory "/var/lib/postgresql/data" exists but is not empty (contains lost+found).

Root Cause: When AWS provisions and formats a new EBS volume with the ext4 filesystem, it automatically injects a lost+found directory. PostgreSQL requires a strictly empty directory to initialize a new database and panics if it detects existing files.

Resolution: Injected the PGDATA environment variable into the StatefulSet configuration (/var/lib/postgresql/data/pgdata), forcing the database to initialize inside a clean subfolder, completely bypassing the native OS directory conflict.

3. Zero-Trust Networking Silently Failing
Symptom: Kubernetes NetworkPolicy manifests designed to isolate the order-service were applied successfully, but lateral curl testing from the frontend pod still reached the restricted service.

Root Cause: The default Amazon VPC CNI solely handles IP address management and routing; it does not contain a policy engine to enforce Kubernetes Network Policies.

Resolution: Deployed Calico (Tigera Operator) as a daemonset across the cluster to act as the network policy engine. Calico instantly parsed the declarative rules and enforced packet-level dropping at the Linux kernel layer, successfully securing the microservice perimeter.

4. AWS Load Balancer Webhook Timeout (IMDSv2 Hop Limit)
Symptom: Pushing the Ingress manifest caused the AWS Load Balancer Controller to crash with an ec2imds: GetMetadata context deadline exceeded error.

Root Cause: AWS enforces IMDSv2 for security, which restricts EC2 metadata queries to a network hop limit of 1. Because the controller runs inside a container (Hop 1) inside a Kubernetes network (Hop 2), AWS actively blocked the controller's attempt to dynamically fetch the VPC ID.

Resolution: Bypassed the metadata server entirely by explicitly injecting the vpcId and region parameters into the controller via Helm upgrade, permanently stabilizing the webhook.

5. Surgical IAM Permission Injection
Symptom: The stabilized AWS Load Balancer Controller threw 403 AccessDenied errors when attempting to provision the physical ALB.

Root Cause: The Helm deployment pulled the newest controller version (v2.8+), which requires the newly added elasticloadbalancing:DescribeListenerAttributes API permission, which was missing from the standard v2.7 IAM policy document.

Resolution: Instead of tearing down the entire IAM role, created an inline JSON policy explicitly granting the missing DescribeListenerAttributes and DescribeCapacityReservation actions. Attached the patch directly to the active AmazonEKSLoadBalancerControllerRole via AWS CLI and restarted the controller deployment, successfully yielding a public-facing ALB URL.

6. Terraform State Desync (The Ghost Load Balancer)
Symptom: Executing terraform destroy resulted in a 20-minute hang, ultimately failing with a DependencyViolation on the VPC public subnets.

Root Cause: The AWS Load Balancer Controller provisioned the ALB, Target Groups, and Security Groups outside of Terraform's state file. Terraform could not delete the subnets because the active "Ghost" Load Balancer was still utilizing the Elastic IP addresses.

Resolution: Manually terminated the controller-provisioned ALB and associated Security/Target groups via the AWS Management Console, clearing the dependency blocks and allowing Terraform to cleanly destroy the remaining VPC architecture.

🚦 How to Deploy
Infrastructure Provisioning:

Bash
cd terraform/
terraform init && terraform apply -auto-approve
aws eks update-kubeconfig --region ap-south-1 --name my-eks-cluster
Core Add-ons:
Install ArgoCD, Calico, and AWS Load Balancer Controller via Helm.

GitOps Bootstrap:
Apply the root ArgoCD application to trigger the declarative sync of the microservices and observability stack:

Bash
kubectl apply -f argocd/root-app.yaml
