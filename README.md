# Ephemeral Preview Environments on AWS
### Terraform · GitHub Actions · ECS Fargate · ECR · ALB · VPC · Docker

[![AWS](https://img.shields.io/badge/AWS-Container-orange?logo=amazon-aws)](https://aws.amazon.com/)
[![Terraform](https://img.shields.io/badge/Terraform-IaC-7B42BC?logo=terraform)](https://www.terraform.io/)
[![GitHub Actions](https://img.shields.io/badge/GitHub-Actions-2088FF?logo=github-actions)](https://github.com/features/actions)
[![ECS](https://img.shields.io/badge/Amazon-ECS%20Fargate-yellow?logo=amazon-aws)](https://aws.amazon.com/ecs/)
[![ECR](https://img.shields.io/badge/Amazon-ECR-FF9900?logo=amazon-aws)](https://aws.amazon.com/ecr/)
[![Docker](https://img.shields.io/badge/Docker-Containerized-2496ED?logo=docker)](https://www.docker.com/)

---

## Overview

This project eliminates the shared staging bottleneck. Every pull request automatically provisions its own fully isolated AWS environment — its own VPC, ALB, ECS Fargate task, and security groups — live in under 5 minutes. When the PR is merged or closed, `terraform destroy` runs automatically and removes every resource, leaving zero idle infrastructure and zero idle cost.

This is a pattern used by companies like Vercel, Render, and Railway — implemented here on AWS with Terraform and GitHub Actions. AWS resources are deployed in the **eu-central-1 region**.

---

## Architecture

![Architecture Diagram](architecture/13_ephemeral_preview_architecture.png)

```
git push → GitHub PR Open
      │
      │  GitHub Actions triggers
      ▼
Docker Build + ECR Push
      │
      ▼
Terraform Apply
      │
      ▼
AWS Cloud
└── VPC  10.0.0.0/16
    └── Public Subnet  10.0.1.0/24
        ├── Internet Gateway  ── Entry point to VPC
        ├── Application Load Balancer  ── Routes inbound HTTP traffic
        ├── ALB Target Group  ── Health checks and forwards to ECS
        └── ECS Fargate Task  ── Runs container, reachable via ALB only
              │
              └── IAM Task Role  ── ECR pull + CloudWatch logs
      │
      ▼
Live Preview URL posted to PR

PR Closed / Merged → Terraform Destroy → Zero idle resources
```

---

## Services Used

| Service | Role |
|---|---|
| **Docker** | Packages the application into a portable container image |
| **Amazon ECR** | Private registry storing and versioning the container image |
| **Amazon ECS Fargate** | Serverless container orchestration — runs tasks without managing servers |
| **Application Load Balancer** | Distributes inbound HTTP traffic and enforces health checks |
| **Amazon VPC** | Isolated virtual network provisioned per PR, destroyed on close |
| **AWS IAM** | Least-privilege task execution role for ECR pull and CloudWatch logging |
| **Terraform** | Provisions and destroys all infrastructure as code on each PR event |
| **GitHub Actions** | CI/CD pipeline that triggers apply on PR open and destroy on PR close |

---

## Deployment Flow

**When a PR is opened:**

1. GitHub Actions triggers on the PR open event
2. Docker image is built from application source and pushed to Amazon ECR
3. `terraform apply` provisions VPC, Internet Gateway, subnet, ALB, ECS cluster, ECS service, security groups, and IAM roles
4. The ECS task pulls the image from ECR and starts the container
5. The ALB health check passes and the preview URL becomes live within 5 minutes
6. The preview URL is posted as a comment on the pull request

**When a PR is closed or merged:**

1. GitHub Actions triggers on the PR close event
2. `terraform destroy` runs against the PR-specific environment
3. The ECS service scales to zero and tasks terminate
4. The ALB, VPC, subnets, Internet Gateway, and security groups are removed in dependency order
5. Zero idle resources remain — cost accumulation stops immediately

---

## Project Structure

```
ephemeral-preview-environments-aws/
│
├── architecture/
│   └── 13_ephemeral_preview_architecture.png    # End-to-end architecture overview
│
├── app/
│   ├── Dockerfile                               # Container image build instructions
│   └── index.html                               # Static web application source
│
├── infrastructure/
│   ├── main.tf                                  # ECS, ALB, VPC resource definitions
│   ├── variables.tf                             # Input variables
│   └── outputs.tf                               # ALB DNS output (preview URL)
│
├── .github/
│   └── workflows/
│       └── deploy.yml                           # PR open → apply · PR close → destroy
│
└── README.md
```

---

## Screenshots

### 1. Docker Build
*Docker image built successfully from the application Dockerfile, ready for ECR.*

![Docker Build](screenshots/02_docker_build_success.png)

---

### 2. Docker Image in Amazon ECR
*Container image pushed to the ECR private repository and available for ECS to pull during deployment.*

![ECR](screenshots/04_ecr_repository_image.png)

---

### 3. VPC and Networking Created
*VPC, subnet, Internet Gateway, and route table provisioned by Terraform for the PR environment.*

![VPC](screenshots/06_vpc_and_network_created.png)

---

### 4. ECS Cluster Running
*ECS cluster provisioned in eu-central-1 as the deployment environment for Fargate tasks.*

![ECS Cluster](screenshots/07_ecs_cluster_created.png)

---

### 5. ECS Service Running
*ECS Service showing the desired task count active with all tasks reporting a running state.*

![ECS Service](screenshots/08_ecs_service_running.png)

---

### 6. Application Load Balancer Active
*ALB provisioned and active, configured to forward inbound HTTP traffic to the ECS Target Group.*

![ALB](screenshots/10_alb_created.png)

---

### 7. Live Preview Environment
*Application accessible through the ALB public DNS endpoint — end-to-end deployment confirmed.*

![Live App](screenshots/11_application_live_browser.png)

---

### 8. Terraform Destroy — Clean Teardown
*All resources successfully destroyed on PR close, confirming zero idle infrastructure remains.*

![Destroy](screenshots/12_terraform_destroy_cleanup.png)

---

## Troubleshooting

### 1. ECS Tasks Failing to Start Due to Port Mismatch

After `terraform apply` completed and the ECS service was created, every task was immediately stopping with a health check failure. The ALB was returning 502 errors and the target group showed all targets as unhealthy, despite the ECS task itself briefly entering a running state before being replaced.

The root cause was a mismatch between the container port defined in the ECS task definition and the port configured on the ALB target group. The task definition was mapping the container to port 3000 — the port the application was bound to — but the target group was probing port 80. The health check received no response on port 80, marked every target unhealthy, and ECS recycled the tasks in a continuous loop.

**Fix:** Updated the container port mapping in the Terraform task definition resource to port 80 and confirmed the target group port matched. Ran `terraform apply` to update the task definition revision, redeployed the service, and confirmed targets moved to healthy within the first two check intervals.

**Lesson:** Container port, target group port, and the application's actual bind port must all be consistent. When tasks reach a running state briefly before failing health checks, a port mismatch is the most likely cause and takes seconds to verify. Check all three values before investigating anything else.

---

### 2. Application Not Reachable via ALB DNS After Deployment

With ECS tasks running and the target group reporting healthy targets, the ALB DNS endpoint was still returning connection timeouts from the browser. The ECS console showed tasks in a running state and the target group health checks were passing, so the issue was not at the application or health check layer.

The cause was a missing inbound rule on the ECS task security group. The ALB was routing requests to the ECS tasks, but the security group attached to the tasks had no rule permitting inbound traffic on port 80 — all inbound connections were being silently dropped at the network layer before reaching the container.

**Fix:** Added an inbound rule to the ECS security group in `main.tf` allowing port 80 from the ALB security group ID rather than from `0.0.0.0/0`. This both fixed the connectivity issue and enforced least-privilege network access — ECS tasks are now reachable only through the load balancer, not directly from the internet.

**Lesson:** A healthy target group does not mean traffic is reaching the container — it means the ALB's health check probe succeeded. Inbound application traffic follows a separate path and requires its own security group rule. Always verify the ECS security group has an explicit inbound rule from the ALB security group before concluding the issue lies elsewhere.

---

### 3. Terraform Destroy Hanging Mid-Execution

During PR close, the GitHub Actions destroy job was hanging for over 10 minutes before eventually timing out. Terraform had begun the destroy sequence but stalled after removing some resources, with the plan showing the ALB and ECS service both pending deletion simultaneously.

The cause was Terraform attempting to delete the ALB and ECS service in parallel. AWS requires ECS tasks to deregister from the target group and terminate before the ALB can be deleted cleanly. Without explicit dependency ordering in the Terraform configuration, Terraform chose a parallel deletion order that triggered AWS-side dependency errors, causing the destroy to block waiting for resource states that would never resolve.

**Fix:** Added an explicit `depends_on` reference in the ALB Terraform resource pointing to the ECS service, ensuring the ECS service scales to zero and all tasks terminate before ALB deletion is attempted. Subsequent destroy runs completed cleanly in under 4 minutes.

**Lesson:** Terraform's dependency graph is built from resource references in the configuration. If two resources don't directly reference each other, Terraform has no way to infer their deletion order and will attempt to remove them in parallel. When `terraform destroy` hangs, check whether the stalled resources have an implicit AWS dependency that isn't expressed in the Terraform configuration, and add `depends_on` to enforce the correct sequence.

---

### 4. ECR Image Not Found When ECS Task Starts

After a new PR triggered the GitHub Actions pipeline and `terraform apply` completed, the ECS tasks were failing immediately with a `CannotPullContainerError`. The ECR repository existed and the image had been pushed successfully in the previous pipeline step, but ECS was unable to locate it.

The cause was a hardcoded image URI in the ECS task definition that referenced an outdated ECR repository tag. The pipeline was building and pushing a new image tagged with the PR number on each run, but the task definition still referenced a static tag from an earlier manual deployment. Terraform applied the task definition without error because the URI string was syntactically valid — it just pointed to an image tag that no longer existed.

**Fix:** Replaced the hardcoded image URI in the task definition Terraform resource with a reference to a Terraform output variable populated dynamically during the ECR push step of the pipeline. This ensured the task definition always referenced the exact image URI that was just pushed, with no manual coordination required.

**Lesson:** Hardcoded image URIs in task definitions are a reliability risk in automated pipelines — any mismatch between what was pushed and what the task definition references causes a silent deployment failure. Passing the ECR URI as a Terraform variable from the build step eliminates the entire class of error and makes the pipeline self-consistent by construction.

---

## What I Learned

This project demonstrated that the value of ephemeral infrastructure comes not just from the automation, but from the discipline it imposes: if your infrastructure can't be destroyed cleanly and recreated identically, it isn't production-ready.

**Automation removes entire categories of human error.** Every manual step in a deployment is a potential failure point — the wrong tag, a missed config update, a forgotten cleanup. This pipeline has zero manual steps from PR open to live environment. The destroy step is as important as the apply step; an environment that can't be torn down reliably will eventually accumulate orphaned resources that become invisible to Terraform and impossible to track.

**Security groups enforce least privilege at the network layer.** The ECS tasks in this project are never directly reachable from the internet — all traffic routes through the ALB, and the ECS security group only accepts inbound connections from the ALB security group. This is a pattern that costs nothing to implement and eliminates an entire attack surface.

**Dependency graphs only become visible when you reverse them.** Building the infrastructure taught how the services connect. Destroying it — and debugging the hanging destroy — taught how they depend on each other at the AWS resource level. The `depends_on` fix required understanding not just what Terraform does during apply, but what AWS enforces during deletion. The debugging process revealed more about service interdependencies than the initial build did.

**Cost and architecture are inseparable design decisions.** Ephemeral environments cost nothing when idle because idle state is not a valid state — the environment either exists for a PR or it doesn't exist at all. This constraint forced cleaner architecture: every resource is justified, nothing persists unnecessarily, and the cost model is transparent and predictable.

---

## Production Improvements

This project uses a single-AZ public subnet architecture optimised for simplicity and low cost as a preview environment. A production system would include:

- **Private subnets** for ECS tasks with a NAT Gateway for outbound traffic
- **Multi-AZ deployment** with the ALB spanning two availability zones for high availability
- **HTTPS** via ACM certificate and Route 53 wildcard DNS per environment
- **Secrets Manager** for application secrets rather than environment variables
- **ECS Service Auto Scaling** based on CPU and memory thresholds
- **CloudWatch dashboards and SNS alarms** for operational monitoring per environment
- **Shared NAT Gateway** across environments to reduce the ~$32/month per-NAT cost at scale

---

## Cost Model

| Resource | Cost when active | Cost when idle |
|---|---|---|
| **ECS Fargate** (0.25 vCPU / 0.5 GB) | ~$0.01/hr | $0.00 |
| **Application Load Balancer** | ~$0.008/hr + LCU | $0.00 |
| **VPC / IGW / Subnets** | $0.00 | $0.00 |
| **ECR image storage** | ~$0.001/GB/month | ~$0.001/GB/month |

**Estimated cost per environment:** ~$0.02–0.05/hour. **Cost when no PRs are open:** ~$0.00.

The only ongoing cost is ECR image storage (~$0.10/month for a typical image). Everything else is destroyed with the environment.

---

## Author

**Sergiu Gota**
AWS Certified Solutions Architect – Associate · AWS Cloud Practitioner

[![GitHub](https://img.shields.io/badge/GitHub-sergiugotacloud-181717?logo=github)](https://github.com/sergiugotacloud)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-sergiu--gota--cloud-0A66C2?logo=linkedin)](https://linkedin.com/in/sergiu-gota-cloud)

> Built as part of a cloud portfolio to demonstrate per-PR ephemeral infrastructure with full automation on AWS.
> Feel free to fork, adapt, or reach out with questions.
