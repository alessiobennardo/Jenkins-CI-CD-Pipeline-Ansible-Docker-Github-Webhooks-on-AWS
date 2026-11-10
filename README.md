# Automated CI/CD Pipeline with Jenkins, Ansible, Docker & GitHub Webhooks on AWS

<img width="1123" height="483" alt="cicd pipeline ansible" src="https://github.com/user-attachments/assets/ac04c51a-40e7-41ef-a4fd-79c823e40028" />


## 📌 Project Overview

This repository demonstrates the end-to-end implementation of an automated Continuous Integration and Continuous Deployment (CI/CD) pipeline.

The pipeline is triggered automatically via **GitHub Webhooks** upon code pushes. **Jenkins** pulls the source code, builds the container image using **Docker**, and delegates the deployment process to **Ansible**, which manages deployment across target **AWS EC2 instances**.

## 🏗 System Architecture & Workflow

### Detailed Workflow:

1. **Source Control Management**: Code change is committed and pushed from local machine to the GitHub repository.

2. **Triggering**: A GitHub Webhook notifies Jenkins instantly upon new commits on the specified branch (Auto-Pull).

3. **Integration & Build (CI)**:

   * Jenkins triggers continuous delivery steps and builds the Docker container image.

4. **Deployment Automation (CD)**:

   * Jenkins triggers **Ansible** playbooks.

   * Ansible connects to target server nodes, manages configurations, and deploys the newly built Docker container to the target environment.

## 🛠️ Prerequisites & Infrastructure Setup

### 1. AWS EC2 Instances Setup

Create two (or more) Ubuntu/Amazon Linux EC2 instances:

* **Jenkins Server & Ansible Control Node**:

  * Type: `t2.micro` or `t2.medium`

  * Security Group: Allow SSH (Port 22), HTTP (Port 80), Jenkins (Port 8080).

* **Target Deployment Server**:

  * Type: `t2.micro`

  * Security Group: Allow SSH (Port 22), HTTP (Port 80 / App Port).

### 2. Software Requirements

On the **Jenkins/Ansible Server**:

```
# Update System
sudo apt update -y && sudo apt upgrade -y

# Install Java (Required for Jenkins)
sudo apt install openjdk-21-jdk -y

# Install Jenkins
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] http://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins -y
sudo systemctl enable --now jenkins

# Install Docker
sudo apt install docker.io -y
sudo usermod -aG docker jenkins
sudo usermod -aG docker $USER
sudo systemctl restart docker

# Install Ansible
sudo apt install ansible -y

```

On the **Target Deployment Server**:

```
# Install Docker on the target machine
sudo apt update -y
sudo apt install docker.io -y
sudo systemctl start docker

```

## 🔑 SSH & Authentication Configuration

Ansible connects to target nodes using SSH. Set up passwordless SSH access from the Ansible Control Node to the Deployment Server:

1. **Generate SSH Key on Jenkins/Ansible Server**:

   ```
   sudo su - jenkins
   ssh-keygen -t rsa -b 4096
   
   ```

2. **Copy Public Key to Target Server**:

   ```
   ssh-copy-id ubuntu@<TARGET_EC2_PRIVATE_OR_PUBLIC_IP>
   
   ```

3. **Verify SSH Connection**:

   ```
   ssh ubuntu@<TARGET_EC2_PRIVATE_OR_PUBLIC_IP>
   
   ```

## 📂 Configuration Files

### 1. Ansible Inventory (`hosts`)

Create an inventory file on your Ansible node (e.g., `/etc/ansible/hosts` or local project folder):

```
[dockerservers]
target_server ansible_host=<TARGET_EC2_IP> ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa

```

### 2. Ansible Deployment Playbook (`deploy.yml`)

```
---
- name: Build and deploy docker container
  hosts: dockerservers
  gather_facts: false
  remote_user: root
  tasks:
    - name: Copia il file sul server remoto
      ansible.builtin.synchronize:
        src: "/var/lib/jenkins/workspace/ansible-jenkins-pipeline/"
        dest: "~/project"

    - name: Build docker image
      community.docker.docker_image:
        name: mico
        tag: latest
        source: build
        build:
          path: "~/project"
        state: present

    - name: Stop container
      community.docker.docker_container:
        name: mico-container
        image: mico:latest
        state: stopped

    - name: Eliminazione container
      community.docker.docker_container:
        name: mico-container
        image: mico:latest
        state: absent

    - name: Creazione container
      community.docker.docker_container:
        name: mico-container
        image: mico:latest
        pull: false
        state: started
        restart_policy: always
        published_ports:
          - "80:80"

```

### 3. Pipeline Script (`Jenkinsfile`)

```
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'my-app-image:latest'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/your-username/your-repo.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo 'Building Docker Image...'
                    sh "docker build -t ${DOCKER_IMAGE} ."
                }
            }
        }

        stage('Run Ansible Playbook for Deployment') {
            steps {
                script {
                    echo 'Executing Deployment via Ansible...'
                    sh 'ansible-playbook -i hosts deploy.yml'
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline executed and deployment completed successfully!'
        }
        failure {
            echo 'Pipeline execution failed. Please check logs.'
        }
    }
}

```

## 🌐 GitHub Webhook Configuration

1. Navigate to your GitHub Repository > **Settings** > **Webhooks** > **Add webhook**.

2. Set **Payload URL**: `http://<YOUR_JENKINS_EC2_PUBLIC_IP>:8080/github-webhook/`

3. Set **Content type**: `application/json`

4. Select event trigger: **Just the push event**.

5. Click **Add webhook**.

## 🚀 Running the Project

1. Open Jenkins Dashboard at `http://<JENKINS_EC2_PUBLIC_IP>:8080`.

2. Create a new **Pipeline** or **Freestyle Project**.

3. Under **Build Triggers**, check **GitHub hook trigger for GITScm polling**.

4. Attach your `Jenkinsfile` or define the build stages.

5. Push a commit to your GitHub repository to observe automatic triggering and deployment.

## 📈 Key Takeaways & Skills Demonstrated

* **Automated CI/CD**: End-to-end automation from code push to production deployment.

* **Infrastructure as Code & Configuration Management**: Automated container lifecycle management using Ansible playbooks.

* **Containerization**: Application packaging using Docker for environmental consistency.

* **Cloud & Security**: AWS EC2 setup, Security Group configurations, and secure SSH key exchange.
