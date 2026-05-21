# Coworking Space Analytics - Deployment Guide

This repository contains the deployment setup for the Coworking Space Analytics API. I used a containerized approach to run a Python (Flask) application on an AWS EKS (Kubernetes) cluster, connected to a PostgreSQL database.

## How the Pipeline Works

Instead of deploying things manually, I set up a simple CI/CD pipeline to handle the heavy lifting:

1. **Database:** I use Bitnami's PostgreSQL Helm chart. It is a reliable way to deploy the DB inside the cluster without managing raw persistent volumes and stateful sets manually.
2. **Docker & ECR:** I containerized the Python application using a Dockerfile. Images are hosted securely in my AWS ECR repository.
3. **AWS CodeBuild:** Whenever I push new code to the repository, AWS CodeBuild automatically triggers. It builds the Docker image, tags it with a semantic version (e.g., `1.0.0`), and pushes it directly to ECR.
4. **Kubernetes:** I deployed the application via standard Kubernetes manifests (`Deployment`, `Service`, `ConfigMap`).

## Releasing a New Build

Here is the exact process to release a new version of the app to the cluster:

1. Make code changes and test them locally.
2. Open `buildspec.yml` and bump the `IMAGE_TAG` variable to the next semantic version (e.g., `1.0.1`).
3. Commit and push the changes to GitHub. (CodeBuild will automatically build and push the new image to ECR).
4. Update `analytics-deployment.yaml` - change the image tag to match the new version.
5. Apply the changes to the cluster by running:
   `kubectl apply -f analytics-deployment.yaml`

Kubernetes will perform a rolling update, spinning up the new pods and terminating the old ones only after the readiness checks pass, ensuring zero downtime.

---

## Infrastructure Notes & Optimizations (Stand Out Suggestions)

While configuring the cluster, I made a few architectural decisions to keep things stable and cost-effective:

### 1. Pod Resource Allocation

I added specific CPU and memory limits to the Kubernetes deployment (`requests: 256Mi RAM / 250m CPU` and `limits: 512Mi RAM / 500m CPU`). Flask applications are fairly lightweight. Setting these limits ensures that if the app ever experiences a memory leak, it won't hog the entire node and crash other services on the cluster.

### 2. Ideal AWS Instance Type

For the EKS node group running this app, **`t3.medium`** (or `t3.small`) EC2 instances are the best fit. Business analytics tools usually experience spiky, unpredictable traffic (e.g., generating heavy reports at the end of the month, but doing little to nothing at night). The `t3` family provides burstable CPU performance, which perfectly handles these short spikes while keeping baseline costs very low compared to `m5` or `c5` instances.

### 3. Strategies for Saving Costs

Cloud costs can grow quickly if left unchecked. Here are a few ways I plan to cut costs for this deployment in the future:

- **Log Retention:** By default, AWS CloudWatch retains logs indefinitely. I should set a retention policy (e.g., 7 or 14 days) on my Container Insights log groups so I don't pay for storing old logs that will never be read.
- **Spot Instances:** I could move my analytics worker pods to AWS Spot Instances, which can be up to 90% cheaper than On-Demand instances.
- **Cluster Autoscaler:** I can configure the cluster to automatically scale down nodes at night or during the weekend when the coworking space is closed and analytics aren't being actively requested.


### 4. Troubleshooting CloudWatch Logs
Setting up Container Insights presented an interesting IAM challenge. Initially, the CloudWatch agent pods were failing with `AccessDeniedException` and `NoCredentialProviders` errors, meaning no log groups were being created in AWS. 

After investigating the agent logs directly via `kubectl`, I realized the root cause: while I configured the CloudWatch Observability add-on to use "EKS Pod Identity" for permissions, the cluster was missing the core **Amazon EKS Pod Identity Agent** add-on required to actually pass those credentials to the pods. Once I installed the missing Pod Identity Agent via the EKS console and restarted the CloudWatch pods (`kubectl delete pods -n amazon-cloudwatch --all`), they successfully authenticated and immediately started streaming my application logs to the CloudWatch log group.