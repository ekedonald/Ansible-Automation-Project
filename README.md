# Ansible Automation Project
## Ansible Client as a Jump Server (Bastion Host)
A [Jumo Server](https://en.wikipedia.org/wiki/Jump_server) (sometimes also referred as [Bastion Host](https://en.wikipedia.org/wiki/Bastion_host)) is an intermediary server through which access to internal network can be provided. If you think about the current architecture you are working on, ideally the Web Servers would be inside a secured network which cannot be reached directly from the Internet. That means, even DevOps engineers cannot `SSH` into the Web Servers directly and can only access it through a Jump Server - it provides better security and reduces [attack surface](https://en.wikipedia.org/wiki/Attack_surface).

In the diagram below, the Virtual Private Network (VPC) is divided into two subents (i.e. Public Subnet has Public IP Addresses and Private Subnet is only reachable by Private IP addresses).

## How to Install and Cofigure Ansible Client to act as a Jump Server/Bastion Host and Create a simple Ansible Playbook to automate server configuration

### Prerequistes
1. Jenkins Server
2. NFS Server
3. Web Server 1
4. Web Server 2
5. Load Balancer
6. Database Server

The following steps are taken to install and configure **Ansible Client** as a **Jump Server/Bastion Host** and create a simple Ansible Playbook to automate servers configuration:

### Step 1: Install and Configure Ansible on an EC2 Instance

* Create a new repository called `ansible-config-mgt` in your GitHub account.

* Update the `Name` tag on your `Jenkins` EC2 Instance to `Jenkins-Ansible`. This server will be used to run playbooks.

* Install Ansible on the `Jenkins-Ansible` server.

```sh
sudo apt update && sudo apt install ansible
```

* Check the version of Ansible running on your instance by running the following command:

```sh
ansible -- version
```

### Step 2: Configure Jenkins Build Job to archive your repository every time changes are made

* Log into Jenkins.

```sh
http://public_ip_jenkins_ansible_instance:8080
```

* Create a new Freestyle Job called `ansible` and point it to your `ansible-config-mgt` repository.

* Configure a Post-Build Job to save all files.

* Configure a webhook in GitHub and set the webhook to trigger `ansible` build. _(**Note**: Trigger Jenkins build exectution only for the main branch.)_

* Test the setup by making changes to the README.md file in the `main` branch and make sure it starts a build automatically and Jenkins saves the files in the order shown below:

```sh
ls /var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/
```

### Step 3: Prepare your development envionment using Visual Studio Code

* 