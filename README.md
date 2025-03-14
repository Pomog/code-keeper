# code-keeper, CI/CD pipeline
 AWS-specific, step-by-step guide that will walk you from zero to a complete DevOps pipeline for the microservices project.

### We’ll create:
1. A GitLab server running on an AWS EC2 instance, installed via Ansible.
2. GitLab Runner (on the same instance or a separate one) to execute pipelines.
3. A Terraform repository to manage Staging and Production AWS resources.
4. Three application repositories (Inventory, Billing, Gateway), each with a pipeline to Build, Test, Scan, Containerize, Deploy (to Staging), get Approval, and then Deploy (to Production).
5. Security best practices (protected branches, secret management, etc.).

### 1. Prepare Your AWS and Local Environment
## 1.1 AWS Account Setup
