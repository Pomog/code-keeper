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
**Important**: In the AWS console, ensure your EC2 Security Group has inbound rules for:

- **SSH (22)**: from your IP address.
- **HTTP (80)**: from 0.0.0.0/0 (or your IP range).
- **HTTPS (443)**: optional, but recommended if you plan to secure GitLab with SSL.

This ensures you can reach GitLab via a browser on port 80 or 443.

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
- Install the Six Library
```bash
sudo apt update
sudo apt install python3-six -y
```

### 4. Automate GitLab Installation via Ansible
#### 4.1 Create Your Ansible Inventory
- On local machine, create a folder `ansible-gitlab` and `inventory.ini`
```
[gitlab-server]
3.127.230.203 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/terraform.pem ansible_python_interpreter=/usr/bin/python3
```
- update ansible locally
```bash
python3 -m venv ansible-env
pip install --upgrade ansible
```
- install Six on the remote host
```bash
ssh -i /home/devops/.ssh/terraform.pem ubuntu@3.127.230.203
sudo apt update
sudo apt install python3-pip -y
sudo pip3 install --upgrade six
python3 -c "import six; print(six.__version__)"
```
- Check Test:
```bash
ansible -i inventory.ini gitlab-server -m ping
```
![image](https://github.com/user-attachments/assets/decbc83e-c378-4032-9ff8-454e8525945d)

#### 4.2 Ansible Playbook: install-gitlab.yml
```bash
vim install-gitlab.yml
```
```
- hosts: gitlab-server
  become: yes
  tasks:
    - name: Install dependencies
      apt:
        update_cache: yes
        name:
          - curl
          - ca-certificates
          - openssh-server
          - postfix
        state: present

    # 1) Add the GitLab GPG key via apt_key module
    - name: Add GitLab repository GPG key
      apt_key:
        url: "https://packages.gitlab.com/gitlab/gitlab-ce/gpgkey"
        state: present

    # 2) Add the GitLab repository for Ubuntu 22.04 (jammy)
    - name: Add GitLab repository
      apt_repository:
        repo: "deb [arch=amd64] https://packages.gitlab.com/gitlab/gitlab-ce/ubuntu jammy main"
        state: present

    - name: Install GitLab
      apt:
        name: gitlab-ce
        state: latest
        update_cache: yes

    - name: Configure external_url
      lineinfile:
        dest: /etc/gitlab/gitlab.rb
        regexp: '^external_url'
        line: "external_url 'http://{{ ansible_default_ipv4.address }}'"

    - name: Reconfigure GitLab
      command: gitlab-ctl reconfigure
```

#### 4.3 Run the Playbook
```bash
cd ansible-gitlab
ansible-playbook -i inventory.ini install-gitlab.yml --list-tasks
ansible-playbook -i inventory.ini install-gitlab.yml
```
![image](https://github.com/user-attachments/assets/b83efdb9-9a9d-47f8-b614-7b082706be87)
![image](https://github.com/user-attachments/assets/0b3f95c1-8c36-4973-a31d-9e43a33d026d)

- Open http://3.127.230.203 in a browser

#### 4.4 Register the GitLab Runner
- Append tasks to install-gitlab.yml
```
- name: Install GitLab Runner
  apt:
    name: gitlab-runner
    state: present
    update_cache: yes
```
- or manually
```bash
sudo apt-get install -y gitlab-runner
```
- Registration
In GitLab (web UI) Admin → Runners find the Registration Token.

- On the server:
```bash
sudo gitlab-runner register \
  --url "http://<YOUR_EC2_ELASTIC_IP>" \
  --registration-token "<TOKEN_HERE>" \
  --executor "shell" \
  --description "my-shell-runner" \
  --tag-list "shell,aws"
```
- Check:
```bash
sudo gitlab-runner status
```

#### 5. Create a Terraform “Infra” Repository
- On local machine, create a folder named infra-repo.
```
infra-repo/
  ├─ .gitignore
  ├─ .gitlab-ci.yml
  ├─ staging/
  │    ├─ main.tf
  │    ├─ variables.tf
  │    └─ outputs.tf
  └─ production/
       ├─ main.tf
       ├─ variables.tf
       └─ outputs.tf
```
- Initialize it as a Git repository
- staging/main.tf
```
provider "aws" {
  region = "us-east-1"  # your AWS region
}

# Simple example: create a single t2.micro instance for staging
resource "aws_instance" "staging_server" {
  ami           = "ami-0c55b159cbfafe1f0"  # Ubuntu 22.04 in us-east-1
  instance_type = "t2.micro"

  tags = {
    Name = "staging-microservice"
  }
}
```
- staging/variables.tf
```
variable "aws_access_key" {}
variable "aws_secret_key" {}

provider "aws" {
  region     = "us-east-1"
  access_key = var.aws_access_key
  secret_key = var.aws_secret_key
}
```
- staging/outputs.tf
```
output "staging_public_ip" {
  value = aws_instance.staging_server.public_ip
}
```
- .gitlab-ci.yml
```
stages:
  - init
  - validate
  - plan
  - apply-staging
  - approve
  - apply-production

variables:
  TF_ROOT: $CI_PROJECT_DIR/staging
  TF_VAR_aws_access_key: $AWS_ACCESS_KEY_ID
  TF_VAR_aws_secret_key: $AWS_SECRET_ACCESS_KEY
  # Adjust region or other variables if needed

init:
  stage: init
  script:
    - cd $TF_ROOT
    - terraform init
  except:
    - tags

validate:
  stage: validate
  script:
    - cd $TF_ROOT
    - terraform validate
  except:
    - tags

plan:
  stage: plan
  script:
    - cd $TF_ROOT
    - terraform plan -out=staging-plan.out
  artifacts:
    paths:
      - staging-plan.out
    when: always
  except:
    - tags

apply-staging:
  stage: apply-staging
  script:
    - cd $TF_ROOT
    - terraform apply "staging-plan.out"
  when: manual
  only:
    - main
    - protected-branch

approve:
  stage: approve
  script:
    - echo "Waiting for manual approval for production..."
  when: manual
  only:
    - main
    - protected-branch

apply-production:
  stage: apply-production
  script:
    - cd production
    - terraform init
    - terraform plan -out=prod-plan.out
    - terraform apply "prod-plan.out"
  when: manual
  only:
    - main
    - protected-branch
```

```bash
git remote add origin
http://<YOUR_EC2_ELASTIC_IP>/root/infra-repo.git
git push -u origin main
```
### 5. Create the Microservice Repositories
- inventory-repo (code from \srcs\inventory-app)
- billing-repo (code from \srcs\billing-app)
- gateway-repo (code from \srcs\api-gateway-app)
#### 5.1 .gitlab-ci.yml
```
stages:
  - build
  - test
  - scan
  - dockerize
  - deploy-staging
  - approve
  - deploy-production

variables:
  DOCKER_IMAGE: "$CI_REGISTRY_IMAGE"
  DOCKER_TAG: "$CI_COMMIT_SHA"

build:
  stage: build
  script:
    - echo "Installing dependencies for build..."
    - pip install -r requirements.txt
  except:
    - tags

test:
  stage: test
  script:
    - echo "Running Python unit tests..."
    - python -m unittest discover
  except:
    - tags

scan:
  stage: scan
  script:
    - echo "Performing security scanning..."
    # e.g. if using Snyk:
    # - pip install snyk
    # - snyk auth $SNYK_AUTH_TOKEN
    # - snyk test --file=requirements.txt
  except:
    - tags

dockerize:
  stage: dockerize
  script:
    - echo "Building and pushing Docker image..."
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
    - docker build -t $DOCKER_IMAGE:$DOCKER_TAG .
    - docker push $DOCKER_IMAGE:$DOCKER_TAG
  except:
    - tags

deploy-staging:
  stage: deploy-staging
  environment:
    name: staging
  script:
    - echo "Deploying to staging environment..."
    # Example approach: use an Ansible or Terraform step to update container
  when: manual
  only:
    - main
    - protected-branch

approve:
  stage: approve
  script:
    - echo "Awaiting approval to deploy to production..."
  when: manual
  only:
    - main
    - protected-branch

deploy-production:
  stage: deploy-production
  environment:
    name: production
  script:
    - echo "Deploying to production environment..."
    # Possibly run another Terraform or Ansible script to pull the new Docker image
  when: manual
  only:
    - main
    - protected-branch
```
#### 5.2 Commit & Push

### 6. Ansible for the Deploy Stage
- deploy-staging.yml

### 7. Testing Everything End-to-End
1. **Ansible**: Re-run `ansible-playbook install-gitlab.yml --list-tasks`. No errors? Great.
2. **GitLab UI**: Log in, create a sample project or open your microservice repos.
3. **Terraform**: In `infra-repo`, push to `main`. See "init, validate, plan" pass. Manually "apply-staging" → AWS has new staging resources. Then "approve" + "apply-production" → production resources are created.
4. **Microservice CI**: In, e.g., `inventory-repo`, push a code change → watch pipeline. 
   - Build → Test → Scan → Dockerize → Deploy to Staging. 
   - Confirm staging environment runs the new image. 
   - Approve → Deploy to Production. 
   - Confirm production environment is updated.
