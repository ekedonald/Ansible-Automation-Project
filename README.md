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
sudo apt update && sudo apt install ansible -y
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

* Create a new Freestyle Job called `ansible`, select **discard old builds** then give it a maximum of 2 builds to keep and point it to your `ansible-config-mgt` repository.

* Select GitHub hook trigger for GitScm polling and configure a Post-Build Job to save all files then click on apply and save.

### Step 3: Configure a webhook in GitHub and set the webhook to trigger `ansible` build

* Go to the `ansible-config-mgt` repository on your GitHub account and click on settings.

* Click on the webhooks tab.

* Click on `Add Webhook`

* Input your password.

* In the **Payload URL**, paste the following URL shown below and click on the **Add Webhook** button:

```sh
http://private_ip_address_jenkins_ansible_server:8080/github-webhook/
```

* Test the setup by making changes to the README.md file in the `main` branch and make sure it starts a build automatically and Jenkins saves the files in the order shown below:

```sh
ls /var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/
```

### Step 4: Prepare your development envionment using Visual Studio Code

* Download and install VS Code, you can get it [here](https://code.visualstudio.com/download)

* After successfully installing VS Code, configure it to connect to your newly created GitHub repository.

* Clone down the `ansible-config-mgt` repository to your local machine.

```sh
git clone <ansible-config-mgt-repository-link>
```

* Go into the `ansible-config-mgt` directory and pull the repo to ensure the directory is up to date.

```sh
cd ansible-config-mgt && git pull
```

### Step 5: Begin Ansible development

* Create a new branch in the `ansible-config-mgt` repository that will be used for development of a new feature using the command shown below:

```sh
git checkout -b prj-145
```

_Note that running the above command will create a new branch and switch to the new branch._

* Create a `playbooks` (i.e. used to store all the playbook files) and `inventory` (i.e. used to keep your hosts organized) directory.

```sh
mkdir playbooks inventory
```

* Create your first playbook named `common.yml` in the playbooks directory.

```sh
cd playbooks && touch common.yml
```

* Create inventory files for each environment (i.e. Devlopment, Staging, Testing and Production) in the inventory directory.

```sh
cd .. && cd inventory && touch dev staging uat prod
```

### Step 6: Set up an Ansible Inventory

An Ansible inventory file defines the hosts and groups of hosts upon which commands, modules and tasks in a playbook operate. Since our intention is to execute Linux commands on remote hosts and ensure that it is the intended configuration particular server that occurs. It is important to have a way to organize our hosts in such an Inventory.

Note: Ansible used TCP port 22 by default which means it needs to `ssh` into target servers from `Jenkins-Ansible` Server. To achieve this, implement the concept of [ssh-agent](https://smallstep.com/blog/ssh-agent-explained/#:~:text=ssh%2Dagent%20is%20a%20key,you%20connect%20to%20a%20server.&text=It%20doesn't%20allow%20your%20private%20keys%20to%20be%20exported.).

The following steps are taken to setup SSH agent and connect VS Code yo your `Jenkins-Ansible`:

1. On your VS Code terminal, run the following command to start up the `ssh-agent`:

```sh
eval `ssh-agent -s`
```

2. Run the following command to add the private key (i.e. keypair used to create the Jenkins-Ansible Instance) to the ssh-agent:

```sh
ssh-add <path_to_the_private_key_of_jenkins_ansible_instance>
```

3. Confirm that the key has been added to the ssh-agent using the command shown below:

```sh
ssh-add -l
```

4. Now, ssh into your `Jenkins-Ansible` server using ssh-agent.

```sh
ssh -A ubuntu@public_ip_address_of_jenkins_ansible
```

Note that your Load Balancer server user is `ubuntu` while the Database, Web and NFS Server's user is `ec2-user` since they are **RHEL-based servers**.

