# CI/CD Pipeline Setup with Jenkins, Maven, Ansible, Docker, and Kubernetes

This repository outlines the steps to set up a complete CI/CD pipeline using **Jenkins**, **Maven**, **Ansible**, **Docker**, and **Kubernetes (EKS)** for deploying applications automatically.

## 1. Install and Configure Jenkins on EC2
Follow the steps below to install and configure Jenkins on an EC2 instance.

1. **Launch an EC2 Instance**:
    - Choose **Amazon Linux 2** and create a new key pair. Keep everything as default. After launching, access the instance via **MobaXterm** using the public IP.

2. **Update and Install Dependencies**:
    ```bash
    sudo su
    cd ~
    sudo yum update -y
    sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
    sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
    sudo yum upgrade -y
    sudo amazon-linux-extras install epel -y
    sudo amazon-linux-extras install java-openjdk11 -y
    sudo yum install java-11-amazon-corretto -y
    sudo yum install jenkins -y
    sudo systemctl enable jenkins  # Enable Jenkins to start on boot
    sudo systemctl start jenkins   # Start Jenkins service
    ```

3. **Verify Java Version**:
    ```bash
    java -version
    javac -version
    ```

4. **Configure Jenkins Hostname**:
    ```bash
    sudo hostnamectl set-hostname Jenkins-Server
    cd /etc
    vim hostname
    # Change hostname to Jenkins-Server
    sudo init 6
    ```

5. **Configure Security Group**:
    - Add inbound rule: **Custom TCP 8080** for **Anywhere (0.0.0.0/0)**.

6. **Access Jenkins**:
    - Open **http://<public-ip>:8080** in your browser.
    - Retrieve Jenkins' initial admin password:
    ```bash
    sudo cat /var/lib/jenkins/secrets/initialAdminPassword
    ```

## 2. Install and Configure Maven

1. **Install Maven**:
    ```bash
    sudo su
    cd ~
    wget https://dlcdn.apache.org/maven/maven-3/3.9.4/binaries/apache-maven-3.9.4-bin.tar.gz
    tar -xzf apache-maven-3.9.4-bin.tar.gz
    mv apache-maven-3.9.4 maven
    ```

2. **Set Maven Environment Variables**:
    - Edit **.bash_profile** to include the following:
    ```bash
    M2_HOME=/opt/maven
    M2=/opt/maven/bin
    JAVA_HOME=/usr/lib/jvm/java-11-openjdk-11.0.23.0.9-2.amzn2.0.1.x86_64
    PATH=$PATH:$HOME/bin:$JAVA_HOME:$M2_HOME:$M2
    export PATH
    ```

3. **Verify Maven Installation**:
    ```bash
    mvn -v
    ```

4. **Configure Jenkins with Maven**:
    - Install **Maven Integration** plugin in Jenkins.
    - Go to **Manage Jenkins** > **Global Tool Configuration**.
    - Add **JDK** and **Maven**:
        - **JDK Name**: `java11`, **JAVA_HOME**: `/usr/lib/jvm/java-11-openjdk-11.0.23.0.9-2.amzn2.0.1.x86_64`
        - **Maven Name**: `maven`, **MAVEN_HOME**: `/opt/maven`

## 3. Set Up Ansible Server and Installation

1. **Launch an EC2 Instance** for Ansible Server:
    - Open port 8080-9090 in security group and access using **MobaXterm**.

2. **Create a New User for Ansible**:
    ```bash
    sudo su
    useradd Bhanu
    passwd Bhanu
    visudo
    # Add line: bhanu ALL=(ALL) NOPASSWD: ALL
    ```

3. **Enable Passwordless Authentication**:
    ```bash
    sudo nano /etc/ssh/sshd_config
    # Change PasswordAuthentication to yes
    sudo service sshd reload
    ```

4. **Install Ansible**:
    ```bash
    sudo amazon-linux-extras install ansible2
    ansible --version
    ```

## 4. Integrate Ansible with Jenkins

1. **Configure Jenkins to Use Ansible**:
    - Go to **Manage Jenkins** > **Configure System** > **Publish over SSH**.
    - Add **Ansible-Server**'s public IP and **bhanu** credentials.

2. **Create Jenkins Job** to deploy artifacts using Ansible:
    - Go to **New Item** > **Maven Project** > Configure.
    - Add **Post-build Action**: "Copy artifacts from workspace" and define the remote directory `/opt/Docker`.

## 5. Install and Configure Docker on Ansible Server

1. **Install Docker** on Ansible Server:
    ```bash
    sudo yum install docker -y
    sudo service docker start
    sudo usermod -aG docker Bhanu
    ```

2. **Create a Dockerfile**:
    ```bash
    nano Dockerfile
    FROM tomcat:latest
    RUN cp -R /usr/local/tomcat/webapps.dist/* /usr/local/tomcat/webapps
    COPY ./*.war /usr/local/tomcat/webapps
    ```

3. **Build Docker Image**:
    ```bash
    docker build -t registerapp:latest .
    ```

## 6. Create Ansible Playbook for Docker Image

1. **Create Playbook** `registerapp.yml`:
    ```yaml
    ---
    - hosts: ansible
      tasks:
        - name: Create Docker image
          command: docker build -t registerapp:latest .
        - name: Create tag to push image to Docker Hub
          command: docker tag registerapp:latest bhanusri772/registerapp:latest
        - name: Push Docker image
          command: docker push bhanusri772/registerapp:latest
    ```

2. **Run Ansible Playbook**:
    ```bash
    ansible-playbook registerapp.yml
    docker login
    ```

## 7. Set Up Kubernetes with EKSCTL

1. **Install kubectl and eksctl** on Bootstrap Server:
    - Follow the [EKSCTL installation guide](https://eksctl.io/).
    - Create an IAM role and attach it to your EC2 instance.

2. **Create Kubernetes Cluster**:
    ```bash
    eksctl create cluster --name cicd-cluster --region ap-south-1 --node-type t2.small
    ```

3. **Verify Cluster**:
    ```bash
    kubectl get nodes
    ```

4. **Create Deployment Manifest** for RegisterApp:
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: cicd-cluster-registerapp
      app: registerapp
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: registerapp
      template:
        metadata:
          labels:
            app: registerapp
        spec:
          containers:
          - name: registerapp
            image: bhanusri772/registerapp
            ports:
            - containerPort: 8080
    ```

5. **Create Service Manifest** for LoadBalancer:
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: cicd-cluster-service
      labels:
        app: registerapp
    spec:
      selector:
        app: registerapp
      ports:
        - port: 8080
          targetPort: 8080
      type: LoadBalancer
    ```

## 8. Integrate Ansible with Kubernetes

1. **Create Ansible Playbook** for Kubernetes Deployment:
    ```yaml
    - hosts: kubernetes
      user: root
      tasks:
        - name: Deploy registerapp on Kubernetes
          command: kubectl apply -f registerapp-deployment.yml
        - name: Create service for registerapp
          command: kubectl apply -f registerapp-service.yml
        - name: Restart deployment with updated image
          command: kubectl rollout restart deployment.apps/cicd-cluster-registerapp
    ```

2. **Execute Playbook**:
    ```bash
    ansible-playbook kube_deploy.yml
    kubectl get pods
    ```

## 9. Create Jenkins Deployment Job for Kubernetes

1. **Create Jenkins Job**:
    - **Freestyle Project** > **Post-build Action**: Execute Ansible Playbook `kube_deploy.yml`.

2. **Set Build Triggers** to poll SCM every 5 minutes:
    ```bash
    *****
    ```

3. **Verify CI/CD** by committing changes to your GitHub repository.

---

## Conclusion

By following these steps, I have successfully integrated Jenkins, Maven, Ansible, Docker, and Kubernetes to create a fully automated CI/CD pipeline.

[Register-app document.pdf](https://github.com/user-attachments/files/17648281/Register-app.document.pdf)






