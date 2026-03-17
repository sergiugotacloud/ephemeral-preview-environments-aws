# Ephemeral Preview Environments on AWS

## 1. Overview

This project implements ephemeral preview environments on AWS using Terraform and GitHub Actions.

Each pull request automatically provisions an isolated environment where changes can be tested safely. Once the pull request is closed, the entire infrastructure is destroyed to optimize cost.

This simulates a real-world CI/CD workflow used in modern engineering teams.

---

## 2. Architecture

The system automates the full lifecycle of infrastructure:

Developer → GitHub Pull Request → GitHub Actions → Terraform → AWS

User → Preview URL → Application Load Balancer → ECS → Container

ECR is used to store container images pulled by ECS tasks.

---

## 3. Architecture Diagram

![Architecture](architecture/13_ephemeral_preview_architecture.png)

---

## 4. Technologies Used

AWS ECS (Fargate)  
AWS ECR  
AWS Application Load Balancer  
AWS VPC  
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

### When PR is Closed

1. GitHub Actions triggers again  
2. Terraform runs `destroy`  
3. All AWS resources are deleted  

---

## 7. Real World Use Case

This architecture is commonly used in teams working on web applications.

Example:

A developer submits a feature (e.g. new checkout flow).  
A temporary environment is created automatically.  
QA or stakeholders access the Preview URL and test the feature safely.  
Once approved and merged, the environment is destroyed.

This prevents conflicts and ensures production stability.

---

## 8. Screenshots

### Docker Build

![Docker](screenshots/02_docker_build.png)

### ECS Service Running

![ECS](screenshots/08_ecs_service.png)

### Application Live

![App](screenshots/11_preview_live.png)

### Terraform Destroy

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

## 11. Lessons Learned

Infrastructure must be designed with destruction in mind  
ALB must be correctly linked to target groups  
IAM roles are critical for ECS task execution  
ECR image references must be precise  
Ephemeral environments significantly reduce cost  

---

## 12. Author

Sergiu Gota  

AWS Certified Solutions Architect – Associate  
AWS Certified Cloud Practitioner  

GitHub  
https://github.com/YOUR_USERNAME
