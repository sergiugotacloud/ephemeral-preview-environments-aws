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

Shared staging environments are a pain. PRs block each other, someone always breaks it right before a demo, and cleanup never happens. This project spins up a fully isolated AWS environment per pull request — VPC, ALB, ECS task, the whole thing — and tears it down automatically when the PR closes. Under 5 minutes from push to live URL, zero cleanup needed.

Same pattern Vercel and Railway use. Built on AWS with Terraform and GitHub Actions. eu-central-1.

---

## Architecture

![Architecture Diagram](architecture/13_ephemeral_preview_architecture.png)

```
git push → PR opens
    ↓
GitHub Actions: Docker build + ECR push
    ↓
terraform apply
    ↓
VPC / Subnet / IGW / ALB / ECS Fargate task / Security groups / IAM
    ↓
Preview URL posted to PR as a comment

PR closes → terraform destroy → nothing left running
```

---

## Services

| Service | What it does here |
|---|---|
| Docker | Builds the container |
| ECR | Stores the image |
| ECS Fargate | Runs it, no servers to manage |
| ALB | Handles inbound traffic, health checks |
| VPC | Isolated network per PR, gone when PR closes |
| IAM | Task role — ECR pull + CloudWatch only, nothing else |
| Terraform | Provisions and destroys all of the above |
| GitHub Actions | Triggers apply on PR open, destroy on PR close |

---

## How it runs

**PR opened:**
1. Actions builds the Docker image, pushes to ECR
2. `terraform apply` — VPC, IGW, subnet, ALB, ECS cluster + service, security groups, IAM roles
3. ECS pulls the image, starts the container
4. ALB health check passes, URL goes live (~5 min end-to-end)
5. URL posted as a PR comment

**PR closed:**
1. `terraform destroy` runs
2. ECS service drops to zero, tasks stop
3. ALB, VPC, subnets, IGW — all gone
4. Nothing left running, billing stops

---

## Project structure

```
ephemeral-preview-environments-aws/
├── architecture/
│   └── 13_ephemeral_preview_architecture.png
├── app/
│   ├── Dockerfile
│   └── index.html
├── infrastructure/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── .github/workflows/
│   └── deploy.yml
└── README.md
```

---

## Screenshots

**Docker build**
![Docker Build](screenshots/02_docker_build_success.png)

**ECR — image pushed**
![ECR](screenshots/04_ecr_repository_image.png)

**VPC + networking**
![VPC](screenshots/06_vpc_and_network_created.png)

**ECS cluster**
![ECS Cluster](screenshots/07_ecs_cluster_created.png)

**ECS service running**
![ECS Service](screenshots/08_ecs_service_running.png)

**ALB active**
![ALB](screenshots/10_alb_created.png)

**Live environment**
![Live App](screenshots/11_application_live_browser.png)

**Terraform destroy — clean**
![Destroy](screenshots/12_terraform_destroy_cleanup.png)

---

## Things that broke

### Port mismatch → 502s everywhere

Classic ECS/ALB issue — comes up constantly in real setups, and I hit it here too.

Tasks were starting and dying immediately. ALB showing 502, target group unhealthy. Spent longer on this than I should have.

Task definition had the container on port 3000. Target group was probing port 80. Health check got nothing, marked everything unhealthy, ECS kept recycling tasks in a loop.

Fixed it by aligning all three: container port, target group port, and what the app actually binds to. They all have to match — obvious in hindsight, easy to miss when you're setting things up separately.

---

### Tasks healthy, app unreachable

Target group showed healthy targets. ECS console showed running tasks. Browser just timed out.

The ECS security group had no inbound rule on port 80. Traffic was getting dropped silently before it ever hit the container. This one is easy to miss because the health check succeeds through a different path — it doesn't prove your actual traffic can get through.

Added an inbound rule scoped to the ALB security group (not `0.0.0.0/0`) — fixed it and tightened the network at the same time. ECS tasks are only reachable through the ALB now, not directly.

---

### terraform destroy hanging for 10+ minutes

Destroy job was stalling mid-run. Terraform was trying to delete the ALB and ECS service in parallel. AWS won't let you delete the ALB while ECS tasks are still registered to the target group, so both resources just sat there waiting on each other.

The issue: these two resources don't reference each other directly in Terraform, so Terraform had no way to know the ordering mattered. Added `depends_on` pointing from the ALB to the ECS service. Destroy now finishes in under 4 minutes.

If `terraform destroy` hangs, look for implicit AWS dependencies that aren't expressed in your config.

---

### CannotPullContainerError — image not found

Pipeline ran fine, `terraform apply` completed, ECS tasks failed immediately. ECR repo existed, image was there. ECS just couldn't find it.

Hardcoded image URI in the task definition was pointing to a tag from an earlier manual deploy. Pipeline was tagging images with the PR number on each run, so the old tag was gone. Terraform applied without errors because the URI was syntactically valid — it just referenced something that no longer existed. Silent failure, annoying to trace.

Replaced the hardcoded URI with a Terraform variable populated dynamically from the ECR push step. Task definition now always references exactly what was just pushed.

---

## What I took away from this

The destroy step is as important as the apply. If you can't tear down cleanly and rebuild identically, the infrastructure isn't actually working — you just haven't noticed yet.

The `depends_on` bug was the most interesting one. Building the infrastructure shows you how services connect. Debugging the destroy shows you how they actually depend on each other. I learned more about the ALB/ECS relationship from that hanging destroy than from setting it up in the first place.

On cost: ephemeral environments are essentially free when idle because idle doesn't exist. Either the PR is open and the environment is running, or it isn't. That constraint also keeps the architecture honest — nothing persists without a reason.

---

## What a production version would add

This setup is cheap and simple, but not production-ready.

- Private subnets for ECS, NAT Gateway for outbound
- Multi-AZ ALB
- HTTPS via ACM + Route 53 wildcard DNS per environment
- Secrets Manager instead of env variables
- ECS auto scaling
- CloudWatch + SNS alerts per environment
- Shared NAT Gateway across environments (~$32/month each otherwise, adds up fast)

---

## Cost

| Resource | Active | Idle |
|---|---|---|
| ECS Fargate (0.25 vCPU / 0.5 GB) | ~$0.01/hr | $0.00 |
| ALB | ~$0.008/hr + LCUs | $0.00 |
| VPC / IGW / Subnets | $0.00 | $0.00 |
| ECR storage | ~$0.001/GB/month | ~$0.001/GB/month |

~$0.02–0.05/hour per active environment. Nothing running when no PRs are open (ECR storage aside, ~$0.10/month).

---

## Author

**Sergiu Gota**
AWS Certified Solutions Architect – Associate · AWS Cloud Practitioner

[![GitHub](https://img.shields.io/badge/GitHub-sergiugotacloud-181717?logo=github)](https://github.com/sergiugotacloud)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-sergiu--gota--cloud-0A66C2?logo=linkedin)](https://linkedin.com/in/sergiu-gota-cloud)
