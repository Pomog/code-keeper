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
- Create an IAM User with “Programmatic Access”:
- Attach a policy or custom role that grants only the minimal permissions needed to create resources in your staging/production environment (EC2, Security Groups, etc.).
![image](https://github.com/user-attachments/assets/97867e4a-ae37-42f8-8146-a60285382db6)
- AWS Key Pair for EC2: In the AWS console → EC2 → Key Pairs → Create or import a key pair.
![image](https://github.com/user-attachments/assets/35f11635-f4a5-4302-916d-0cae3ab99f26)

### 2. Local Environment Setup
- we will use ubuntu/jammy64 vagrant box 
```bash
vagrant init uubuntu/jammy64
```
- Install Ansible:
```bash
sudo apt update -y && sudo apt install ansible -y
```
- Install Terraform:
```bash
sudo snap install terraform --classic
```
![image](https://github.com/user-attachments/assets/a7f3e5ca-7321-4062-8274-22512572a1ba)
- AWS CLI
```bash
sudo apt  install awscli -y
```
- Run `aws configure` to store your terraform-user credentials locally
  `aws sts get-caller-identity` → if the AWS CLI is configured properly, it returns your IAM user

### 3. Launch an EC2 Instance for GitLab
- EC2 Dashboard → Launch Instance
- In EC2 → Instances, confirm your instance is running
![image](https://github.com/user-attachments/assets/571c09c4-5a86-47f6-bb29-c517a7e2f1d0)

- Copy the PEM File 
 ![image](https://github.com/user-attachments/assets/9d3f8975-3b93-4b92-b7c6-53f689068386)
- SSH into the instance
```bash
sudo usermod -aG sudo devops
chmod 600 /home/devops/.ssh/terraform.pem
ssh -i /home/devops/.ssh/terraform.pem ubuntu@3.127.230.203
```
![image](https://github.com/user-attachments/assets/0e2385a2-4733-420d-a708-25f2f3b80562)


### 4. Automate GitLab Installation via Ansible


