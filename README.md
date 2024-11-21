
CI/CD Pipeline for Flask Application Deployment on AWS using Docker, Terraform, Ansible, and GitHub Actions
This project demonstrates the deployment of a Flask application using a robust CI/CD pipeline that integrates Terraform, Ansible, Docker, and GitHub Actions. The pipeline automates infrastructure provisioning, configuration management, and application deployment, offering a seamless approach to managing DevOps workflows.

Project Overview
Objective
To create and deploy a containerized Flask application using AWS EC2 instances, orchestrated through a CI/CD pipeline leveraging industry-standard tools:

Infrastructure Provisioning:
Two AWS EC2 instances are provisioned:
Control Node: Amazon Linux for managing Ansible playbooks.
Host Node: Ubuntu for hosting the Dockerized Flask application.
Configuration Management:
Ansible is used to automate the setup of Docker on the host node.
Application Deployment:
GitHub Actions orchestrates the build and deployment of the Flask application, containerized with Docker, and deployed to the host node.
Project Implementation
1. Prerequisites
Ensure the following are set up locally:

AWS CLI configured with valid credentials.
Terraform installed and ready to use.
A GitHub repository containing the Flask application and Dockerfile.
Docker Hub account with credentials for GitHub Actions integration.
Basic understanding of Ansible and YAML.
2. Infrastructure Provisioning Using Terraform
Create infrastructure using a Terraform script (main.tf) to provision:

Amazon Linux EC2 instance (control node) with Ansible pre-installed.
Ubuntu EC2 instance (host node) with Docker installed.
Key Terraform Configuration
hcl
Copy code
resource "aws_instance" "amazon_linux" {
  ami           = "ami-00385a401487aefa4"
  instance_type = "t2.micro"
  # Add additional configurations like security groups, key pairs, and user data
}

resource "aws_instance" "ubuntu" {
  ami           = "ami-0a422d70f727fe93e"
  instance_type = "t2.micro"
  # Add additional configurations like security groups, key pairs, and user data
}
Initialize and Apply Terraform
bash
Copy code
terraform init
terraform apply
3. Configuring the Control Node
SSH into the Control Node:

bash
Copy code
ssh -i <your-key.pem> ec2-user@<ControlNodePublicIP>
Set Up SSH Access to the Host Node:

Copy the private key to the control node.
Test the SSH connection to the host node:
bash
Copy code
ssh -i "your-key.pem" ubuntu@<HostNodePrivateIP>
4. Host Node Configuration with Ansible
Use Ansible to automate the installation and configuration of Docker on the host node.

Ansible Playbook: install_docker.yml
yaml
Copy code
---
- name: Install Docker on the host node
  hosts: all
  become: yes  # Run tasks with root privileges
  tasks:
    - name: Install dependencies
      apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
          - software-properties-common
        state: present
        update_cache: yes

    - name: Add Docker's official GPG key
      apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present

    - name: Set up Docker repository
      apt_repository:
        repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu jammy stable
        state: present

    - name: Install Docker
      apt:
        name: docker-ce
        state: latest
        update_cache: yes

    - name: Enable and start Docker
      systemd:
        name: docker
        enabled: yes
        state: started
Run the Ansible Playbook
bash
Copy code
ansible-playbook install_docker.yml
5. Application Preparation
Flask Application (app.py)

python
Copy code
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    return '<h1>Hello, World! This Flask app is running in a Docker container!</h1>'

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=5000)
Dockerfile

dockerfile
Copy code
FROM python:3.9-slim
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir -r requirements.txt
EXPOSE 5000
CMD ["flask", "run", "--host=0.0.0.0", "--port=5000"]
Push to GitHub Repository

6. CI/CD Pipeline with GitHub Actions
Workflow File: .github/workflows/pipeline.yml
yaml
Copy code
name: CI/CD Pipeline for Flask Application

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Log in to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and push Docker image
      run: |
        docker build -t ${{ secrets.DOCKER_USERNAME }}/flask-app:latest .
        docker push ${{ secrets.DOCKER_USERNAME }}/flask-app:latest

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
    - name: Set up SSH
      uses: webfactory/ssh-agent@v0.7.0
      with:
        ssh-private-key: ${{ secrets.EC2_SSH_PRIVATE_KEY }}

    - name: Deploy to EC2
      run: |
        ssh -o StrictHostKeyChecking=no ubuntu@<HostNodePublicIP> \
        'sudo docker pull ${{ secrets.DOCKER_USERNAME }}/flask-app:latest && \
         sudo docker stop flask-app || true && \
         sudo docker rm flask-app || true && \
         sudo docker run -d --name flask-app -p 5000:5000 ${{ secrets.DOCKER_USERNAME }}/flask-app:latest'
Set GitHub Secrets
DOCKER_USERNAME: Docker Hub username.
DOCKER_PASSWORD: Docker Hub password.
EC2_SSH_PRIVATE_KEY: SSH private key for the EC2 instance.
7. Accessing the Application
Navigate to http://<HostNodePublicIP>:5000 to see the deployed Flask application.
8. Cleanup Resources
To avoid unnecessary costs:

bash
Copy code
terraform destroy
Conclusion
This project demonstrates an automated and scalable approach to deploying a Flask application using a modern CI/CD pipeline. By integrating Terraform, Ansible, Docker, and GitHub Actions, this workflow ensures seamless provisioning, configuration, and deployment, adhering to best practices in DevOps.
