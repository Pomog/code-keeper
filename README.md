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
