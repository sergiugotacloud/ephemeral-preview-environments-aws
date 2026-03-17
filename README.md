# Ephemeral Preview Environments on AWS

## 1. Overview

This project implements ephemeral preview environments on AWS using Terraform and GitHub Actions.

Each pull request automatically provisions an isolated environment where changes can be tested safely. Once the pull request is closed, the entire infrastructure is destroyed to optimize cost.

This simulates a real-world CI/CD workflow used in modern engineering teams, where environments are short-lived and fully automated.

---

## 2. Architecture

The system automates the full lifecycle of infrastructure:

Developer → GitHub Pull Request → GitHub Actions → Terraform → AWS

User → Preview URL → Application Load Balancer → ECS → Container

ECR is used to store container images pulled by ECS tasks.

Key principle:

Each Pull Request creates its own isolated environment, which is destroyed after use.

---

## 3. Architecture Diagram

![Architecture](architecture/13_ephemeral_preview_architecture.png)

---

## 4. Technologies Used

AWS ECS (Fargate)  
AWS ECR  
AWS Application Load Balancer  
AWS VPC (Public Subnet + Internet Gateway)  
AWS IAM  
Docker  
Terraform  
GitHub Actions  

---

## 5. Project Structure

```
ephemeral-preview-environments-aws
├── architecture/
├── app/
├── infrastructure/
├── screenshots/
├── README.md
```

---

## 6. Deployment Workflow

### When PR is Opened

1. GitHub Actions is triggered  
2. Docker image is built  
3. Image is pushed to ECR  
4. Terraform runs `apply`  
5. AWS infrastructure is created  
6. Preview URL becomes available  

Result:  
A fully isolated environment is created for that specific Pull Request.

---

### When PR is Closed

1. GitHub Actions triggers again  
2. Terraform runs `destroy`  
3. All AWS resources are deleted  

Result:  
The environment is completely removed, leaving no unused infrastructure.

---

## 7. Real World Use Case

This architecture is commonly used in teams working on web applications and microservices.

Example workflow:

1. A developer creates a feature branch (e.g. new checkout flow)  
2. A Pull Request is opened  
3. A temporary environment is created automatically  
4. QA or stakeholders access the Preview URL and test the feature  
5. Once approved, the PR is merged  
6. The environment is destroyed automatically  

This prevents:

- Environment conflicts between developers  
- Breaking shared staging environments  
- Deploying untested changes to production  

---

## 8. Screenshots

### Docker Build

![Docker](screenshots/02_docker_build_success.png)

### ECS Service Running

![ECS](screenshots/08_ecs_service_running.png)

### Application Live

![App](screenshots/11_application_live_browser.png)

### Terraform Destroy (Cleanup)

![Destroy](screenshots/12_terraform_destroy_cleanup.png)

---

## 9. Key Concepts Demonstrated

Infrastructure as Code  
Ephemeral environments  
CI/CD automation  
Containerized applications  
AWS networking and load balancing  
Cost optimization through lifecycle management  

---

## 10. Challenges Encountered

### Problem  
ECS tasks were not starting.

### Cause  
Incorrect container port configuration in task definition.

### Solution  
Aligned container port with ALB target group configuration.

---

### Problem  
Application was not accessible via ALB.

### Cause  
Security group did not allow inbound HTTP traffic.

### Solution  
Updated security group to allow port 80 access.

---

### Problem  
Terraform destroy was hanging for several minutes.

### Cause  
ECS service dependencies prevented immediate deletion.

### Solution  
Waited for ECS service to scale down before full cleanup.

---

### Problem  
ECR image not found during deployment.

### Cause  
Incorrect image URI in ECS task definition.

### Solution  
Corrected repository URI and ensured image push completed before deployment.

---

## 11. Failure Scenarios and System Behavior

### ECS Task Failure

If the container fails to start:

- ECS attempts to restart the task automatically  
- ALB health checks fail  
- Traffic is not routed to unhealthy tasks  

Result:  
Service remains unavailable until a healthy task is running.

---

### ALB Health Check Misconfiguration

If health check path or port is incorrect:

- ALB marks all targets as unhealthy  
- No traffic is routed  

Fix:  
Ensure health check matches application endpoint.

---

### ECR Image Missing

If the image is not available in ECR:

- ECS task fails to start  
- Deployment fails  

Fix:  
Ensure image is pushed before Terraform apply.

---

### Terraform Destroy Delays

During destroy:

- ECS service takes time to scale down  
- ALB and networking dependencies delay deletion  

Observation:  
Infrastructure deletion is not instant due to AWS dependency handling.

---

### GitHub Actions Failure

If CI/CD fails:

- No infrastructure is created  
- No preview environment is available  

Fix:  
Check workflow logs and permissions.

---

## 12. Key Engineering Takeaways

- Infrastructure should be treated as temporary, not permanent  
- Automation is essential for scalable environments  
- Systems must be designed for both creation and destruction  
- Observability is required to debug distributed systems  
- Clean architecture improves reliability and maintainability  

---

## 13. Author

Sergiu Gota  

AWS Certified Solutions Architect – Associate  
AWS Certified Cloud Practitioner  
