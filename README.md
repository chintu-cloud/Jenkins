 # ======== Day 01 – Jenkins Setup on AWS EC2 🚀 ========
 
# This guide walks through setting up Jenkins on an EC2 instance, installing required dependencies, and configuring a Jenkins pipeline to run Terraform code.
# 🖥️ EC2 Instance Setup
- Instance Name: Jenkins
- Instance Type: c7i-flex.large      ## for speed access
- OS: Amazon Linux 2
  
# 🔧 Installation Steps
1. Connect to EC2 and Switch to Root
ssh -i <your-key>.pem ec2-user@<public-ip>
sudo su -

# 2. Create Installation Script
Create a shell script install.sh to install all dependencies:
vim install.sh

Paste the following content:

or you can install it one by one

#!/bin/bash
sudo yum update -y

# Git
sudo yum install git -y

# Java (required for Jenkins)
sudo yum install java-17-amazon-corretto.x86_64 -y

# Jenkins
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum install jenkins -y
sudo systemctl enable jenkins
sudo systemctl start jenkins

# Terraform
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
sudo yum -y install terraform

# 3. Make Script Executable and Run
chmod 755 install.sh
sh install.sh

# - Open your browser and navigate to:
 http://public-ip:8080
- Copy the initial admin password from the path shown on the Jenkins page:
cat /var/lib/jenkins/secrets/initialAdminPassword
- Complete the setup wizard and install recommended plugins.

#  🔐 IAM Role for Terraform
Attach an IAM role to the EC2 instance with permissions to manage AWS resources. This is required for Terraform to function properly within Jenkins.

# 🔌 Install Required Jenkins Plugins
- Go to: Manage Jenkins → Plugins
- Install: Pipeline, Pipeline: Stage View
- Return to Jenkins home page

# Configure Jenkins Pipeline
Create a new pipeline job and use the following script:


    pipeline {
      stages {
          stage('Clone Repo') {
            steps {
                 git url: 'https://github.com/chintu-cloud/Terraform_CICD.git'
            }
        }
        stage('Terraform Init') {
            steps {
                dir('day-1-basic-code') {
                    sh 'terraform init'
                }
            }
        }
        stage('Terraform Destroy') {
            steps {
                dir('day-1-basic-code') {
                    sh 'terraform destroy -auto-approve'
                }
            }
        }
    }
    }

# Jenkins Workspace Path
Terraform code will execute from:
cd /var/lib/jenkins/workspace/<pipeline-name>

- Click Build Now to trigger the pipeline.
- Monitor console output for execution logs.





#======== DAY02 JENKINS ========
-------------------------
Connect to Jenkins on EC2
install the dependencies 

# Retrieve the initial Jenkins admin password:
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
# Open Jenkins in browser:
http://public-ip:8080
# 📦 2. Install Required Plugins
Navigate to:
Jenkins → Manage Jenkins → Plugins → Available Plugins
Install the following plugins:

-Pipeline Stage View
-Blue Ocean
-Generic Webhook Trigger
-GitHub Integration

# 🛠️ 3. Create a New Pipeline Job
Optional: Add Build Parameters
Useful for choosing apply or destroy during Terraform operations.
 This project is parameterized
→ Add Parameter → Choice Parameter
Name: action
Choices:
apply
destroy
# Declarative Pipeline Example
    pipeline {
    agent any

    stages {

        stage('Clone Repo') {
            steps {
                 git url: branch: 'main', 'https://github.com/chintu-cloud/Terraform_CICD.git'
            }
        }

        stage('Terraform Init') {
            steps {
                dir('day-1-basic-code') {
                    sh 'terraform init'
                }
            }
        }

        stage('Terraform Plan') {
            steps {
                dir('day-1-basic-code') {
                    sh 'terraform plan'
                }
            }
        }

        stage('Terraform Apply/Destroy') {
            steps {
                dir('day-1-basic-code') {
                    sh "terraform ${params.action} -auto-approve"
                }
            }
        }

    }
    }

# 📜 5. Scripted Pipeline Example
    node {

    stage('Clone Repo') {
        git url: branch: 'main', 'https://github.com/chintu-cloud/Terraform_CICD.git'
    }

    stage('Terraform Init') {
        sh 'terraform init'
    }

    stage('Terraform Plan') {
        sh 'terraform plan'
    }

    stage('Terraform Apply') {
        sh "terraform ${params.button} -auto-approve"
    }

    }

 # ⏱️ 6. Jenkins Trigger Types
 ##  1. Build After Other Projects Are Built
Used for chaining pipelines.
Steps:
Pipeline2 → Configure → Build Triggers → Build after other projects are built
Project: Pipeline1
Pipeline2 will run automatically after Pipeline1 completes.

# 2. Build Periodically (CRON)
\* \* \* \* \*


Meaning → Run the pipeline every 1 minute
Trigger continues until manually disabled.

# 3. GitHub Hook Trigger for GITScm Polling
Automatically runs when changes are pushed to GitHub.
# Jenkins:steps
Build Triggers → GitHub hook trigger for GITScm polling
# GitHub Repository:
Settings → Webhooks → Add Webhook
Payload URL: http://<public-ip>:8080/github-webhook/
Content type: application/json
After saving — committing code will automatically trigger Jenkins.
# 4. Poll SCM Trigger

Combination of webhook + schedule.

Example:
   \* \* \* \* \*

   
Behaviour:

Jenkins does not run immediately after a GitHub commit

It checks for changes according to the schedule (every minute)

# Summary
✔ Connecting Jenkins on EC2
✔ Installing essential plugins
✔ Creating Declarative & Scripted pipelines
✔ Adding Apply/Destroy parameters
✔ Implementing Jenkins triggers:
 • Upstream build trigger
 • Cron-based trigger
 • GitHub webhook
 • Poll SCM trigger

 

# ====== Day03 jankins ======

# Jenkins – Master/Slave Setup, Change Default Jenkins Port & Change Jenkins Home Path

This guide covers three important Jenkins administration concepts:

Jenkins Master–Slave (Agent) Architecture

Changing Jenkins Default Port (8080 → Custom Port)

Changing Jenkins Default Home Directory (/var/lib/jenkins → Custom Path)

🚀 1. Jenkins Master–Slave Architecture

Jenkins Master–Slave (Controller–Agent) allows you to distribute your CI/CD workloads across multiple servers.

✅ Why Use Master–Slave?

Run builds on different OS (Linux, Windows, etc.)

Reduce load on the master node

Scale Jenkins horizontally

Isolate builds for security


========= master -slave concept =========

# lunch 2 instances 
      1. Jenkins-master (server)
      2. slave  (server)
      3. c7i-flex.large (instance type)
      4. key-pair required
      5. Attach IAM role 
      

 <img width="1833" height="583" alt="Screenshot 2025-11-23 104926" src="https://github.com/user-attachments/assets/d4b45add-9209-47d8-bebb-b576bc9432ef" />
 


connect Jenkins-master

             sudo su -
             
             yum install git -y                                          // ------- install git --------
             
             sudo yum install java-17-amazon-corretto.x86_64             // -------java dependency for jenkins-------
             
             sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
             sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
             sudo yum install jenkins -y
             sudo systemctl enable jenkins
             sudo systemctl start jenkins                                 //------ jenkins install-------

             sudo yum install -y yum-utils
             sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
             sudo yum -y install terraform                                // -------install terraform --------
             
             
5. then copy Jenkins-master server public IP & paste in search bar with port no.

     ----  Jenkins port no. = 8080 ----    (access the jenkins server)
   ex:  13.223.100.239:8080

6. then Go to Jenkin server page
         --> unlock Jenkins option their select
              /var/lib/jenkins/secrets/initialAdminpassword
   <img width="1027" height="394" alt="Screenshot 2025-11-22 114328" src="https://github.com/user-attachments/assets/b3dcd895-2560-4999-9f35-545973cf0d3b" />
        (for access copy & paste in inside server with using password)
   using cat command
   <img width="637" height="37" alt="Screenshot 2025-11-22 114307" src="https://github.com/user-attachments/assets/c2b55d6f-e041-4651-8263-374c74573df0" />
   give password in path

7. then create first Admin user
   <img width="1052" height="667" alt="Screenshot 2025-11-22 114537" src="https://github.com/user-attachments/assets/1d0213da-145e-4b87-94f1-b18e58ec26b3" />

8. then create node
      name = node -1
      no. of executors = 1
      Remote root directry = /home/ec2-user/
      level = terraform 
      disk = 200 MB
             200 MB
             1 GB
             2 GB

   
   <img width="336" height="426" alt="Screenshot 2025-11-22 114853" src="https://github.com/user-attachments/assets/ba2b2fe0-5c93-491c-aa6d-510f0dd7c90f" />
   
   <img width="410" height="758" alt="Screenshot 2025-11-22 114943" src="https://github.com/user-attachments/assets/956ca9f4-23dc-46e8-8310-249dd56845ed" />

 
 connect slave 
 
             sudo su -
             yum install git -y                                          // ------- install git --------
             sudo yum install java-17-amazon-corretto.x86_64             // -------java dependency for jenkins-------
             
             sudo yum install -y yum-utils
             sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
             sudo yum -y install terraform                                // -------install terraform --------

   then copy node-1 agent & paste inside in slave server 
    <img width="1432" height="283" alt="Screenshot 2025-11-22 115321" src="https://github.com/user-attachments/assets/06457d22-6ec7-425f-89d5-4ddc3338d0d5" />

   then check node-1 is showing online
   <img width="1920" height="457" alt="Screenshot (444)" src="https://github.com/user-attachments/assets/f5a7e6f7-d136-447d-be98-caadd53080f5" />

   then create New item +
       name = first-pipeline
       install plugins = pipeline: stage view 
       
   then goto configure option 
      pipeline script inside code given

         pipeline {
            agent none   // We will use per-stage agents
            stages {

                     /* -----------------------------
                       GIT CLONE (Runs on any node)
                      ------------------------------*/
             stage('Git Clone') {
                 agent { label 'terraform' }   // OR 'master' if you prefer
                 steps {
                     git url: branch: 'main', 'https://github.com/chintu-cloud/Terraform_CICD.git'
                }
            }
         }
     }

     
     then apply & save  -->  then build now  

     
<img width="1035" height="758" alt="Screenshot (443)" src="https://github.com/user-attachments/assets/2edeea64-39f4-4a84-92a5-474abcd27e23" />
     



again connect Jenkin-server 


 <img width="580" height="419" alt="Screenshot 2025-11-23 084857" src="https://github.com/user-attachments/assets/b316e15b-3dc5-43f0-992b-ec3cd8482621" />

     

# ======= Change port no. =======

🔥 2. Change the Default Jenkins Port (8080 → Another Port)

The Jenkins default port is 8080.
You can change it to 9090, 8000, etc.

Method: Edit Jenkins Service File

Open the systemd service file:

sudo vi /usr/lib/systemd/system/jenkins.service

Find this line:

Environment="JENKINS_PORT=8080"

Change it to:

Environment="JENKINS_PORT=9090"
<img width="771" height="675" alt="Screenshot (445)" src="https://github.com/user-attachments/assets/82b9992b-b231-4505-b3b0-dec20b86733b" />



Restart Jenkins

sudo systemctl daemon-reload

sudo systemctl restart jenkins

sudo systemctl status jenkins




<img width="1639" height="621" alt="Screenshot (446)" src="https://github.com/user-attachments/assets/e026265a-f409-4254-acaf-ac6be52240f1" />


Verify in browser:

http://public-ip:9090
<img width="1675" height="985" alt="Screenshot (447)" src="https://github.com/user-attachments/assets/7eefe31b-6de0-4c62-9999-1411a8ef3ab7" />


📦 3. Change the Default Jenkins Home Path (/var/lib/jenkins)

Default Jenkins home directory:

/var/lib/jenkins


If you want to move Jenkins data to a new location (example: /mnt/jenkins_home):
# stop the jenkins first

systemctl stop jenkins

# 1️⃣ Create New Directory
sudo mkdir -p /mnt/jenkins_home
sudo chown -R jenkins:jenkins /mnt/jenkins_home
sudo chmod 750 /mnt/jenkins_home

# 2️⃣ Update Jenkins Service File

Open:

sudo vi /usr/lib/systemd/system/jenkins.service


Find:

Environment="JENKINS_HOME=/var/lib/jenkins"
WorkingDirectory=/var/lib/jenkins


Change to:

Environment="JENKINS_HOME=/mnt/jenkins_home"
WorkingDirectory=/mnt/jenkins_home

# 3️⃣ Copy Old Jenkins Data
sudo cp -a /var/lib/jenkins/. /mnt/jenkins_home/

# 4️⃣ Restart Jenkins
sudo systemctl daemon-reload
sudo systemctl restart jenkins


Verify 

 ls /mnt/jenkins_home/workspace
