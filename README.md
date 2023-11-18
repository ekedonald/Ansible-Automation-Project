# Ansible Automation Project
## Ansible Client as a Jump Server (Bastion Host)
A [Jump Server](https://en.wikipedia.org/wiki/Jump_server) (sometimes also referred to as [Bastion Host](https://en.wikipedia.org/wiki/Bastion_host)) is an intermediary server through which access to the internal network can be provided. If you think about the current architecture you are working on, ideally the Web Servers would be inside a secured network that cannot be reached directly from the Internet. That means, even DevOps engineers cannot `SSH` into the Web Servers directly and can only access it through a Jump Server - it provides better security and reduces [attack surface](https://en.wikipedia.org/wiki/Attack_surface).

In the diagram below, the Virtual Private Network (VPC) is divided into two subnets (i.e. Public Subnet has Public IP Addresses and Private Subnet is only reachable by Private IP addresses).

## How to Install and Configure Ansible Client to act as a Jump Server/Bastion Host and Create a simple Ansible Playbook to automate server configuration

### Prerequistes
1. Jenkins Server
2. NFS Server
3. Web Server 1
4. Web Server 2
5. Load Balancer
6. Database Server

The following steps are taken to install and configure **Ansible Client** as a **Jump Server/Bastion Host** and create a simple Ansible Playbook to automate server configuration:

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

* Select GitHub hook trigger for GitScm polling and configure a Post-Build Job (i.e. Archive the artifacts) to save all `**` files then click on apply and save.

### Step 3: Configure a webhook in GitHub and set the webhook to trigger Ansible build

* Go to the `ansible-config-mgt` repository on your GitHub account and click on settings.

* Click on the webhooks tab.

* Click on `Add Webhook`

* Input your password.

* In the **Payload URL**, paste the following URL shown below and click on the **Add Webhook** button:

```sh
http://private_ip_address_jenkins_ansible_server:8080/github-webhook/
```

* Test the setup by making changes to the README.md file in the `main` branch and make sure it starts a build automatically as shown below:

### Step 4: Prepare your development environment using Visual Studio Code

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

### Step 5: Begin Ansible development and Set up an Ansible Inventory

* Create a new branch in the `ansible-config-mgt` repository that will be used for the development of a new feature using the command shown below:

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

* Create inventory files for each environment (i.e. Development, Staging, Testing and Production) in the inventory directory.

```sh
cd .. && cd inventory && touch dev staging uat prod
```

An Ansible inventory file defines the hosts and groups of hosts upon which commands, modules and tasks in a playbook operate. Since we intend to execute Linux commands on remote hosts and ensure that it is the intended configuration particular server that occurs. It is important to have a way to organize our hosts in such an Inventory.

* Update your `inventory/dev.yml` file with the code shown below:

```sh
[nfs]
<NFS-Server-Private-IP-Address> ansible_ssh_user='ec2-user'

[webservers]
<Web-Server1-Private-IP-Address> ansible_ssh_user='ec2-user'
<Web-Server2-Private-IP-Address> ansible_ssh_user='ec2-user'

[db]
<Database-Private-IP-Address> ansible_ssh_user='ec2-user' 

[lb]
<Load-Balancer-Private-IP-Address> ansible_ssh_user='ubuntu'
```

### Step 6: Setup an ssh-agent and Connect VS Code to your Jenkins-Ansible Instance

Ansible uses TCP port 22 by default which means it needs to `ssh` into target servers from `Jenkins-Ansible` Server. To achieve this, implement the concept of [ssh-agent](https://smallstep.com/blog/ssh-agent-explained/#:~:text=ssh%2Dagent%20is%20a%20key,you%20connect%20to%20a%20server.&text=It%20doesn't%20allow%20your%20private%20keys%20to%20be%20exported.).

The following steps are taken to setup the ssh-agent and connect VS Code to your `Jenkins-Ansible`:

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

_Note that your Load Balancer server user is `ubuntu` while the Database, Web and NFS Server's user is `ec2-user` since they are **RHEL-based servers**._


### Step 7: Set up an ssh-agent on the Jenkins-Ansible Instance so it will be able to connect to the other servers

* Run the following command to start up the `ssh-agent`

```sh
eval `ssh-agent -s`
```

* Create a file similar to the keypair file used to ssh into your instance and paste the content of the keypair file.

```sh
vi web11.pem
```

* Give write permissions to the file and add the newly created file into the ssh-agent using the commands shown below:

```sh
chmod 400 web11.pem
ssh-add web11.pem
```

_Note that if you used different keypairs when provisioning your servers (i.e. NFS, Database, Load Balancers and Web), you must create files that match the keypairs and add them to the ssh-agent._

### Step 8: Create a common playbook

It is time to start giving Ansible the instructions on what you need to perform on all servers listen in `inventory/dev`. In the `common.yml` playbook, you will write configurations for repeatable, reusable and multi-machine tasks that are common to systems with the infrastructure.

* Update your `playbooks/common.yml` file with the following code:

```sh
---
- name: update web, nfs and db servers
  hosts: webservers, nfs, db
  become: yes
  tasks:
    - name: ensure wireshark is at the latest version
      yum:
        name: wireshark
        state: latest
   

- name: update LB server
  hosts: lb
  become: yes
  tasks:
    - name: Update apt repo
      apt: 
        update_cache: yes

    - name: ensure wireshark is at the latest version
      apt:
        name: wireshark
        state: latest
```

The code above has two plays, the **first play** is dedicated to the **RHEL Servers** (i.e. NFS, Database, Web). The task will be performed as the `root` user hence the use of the **become: yes** key value pair. Since they are all **RHEL Based Servers**, the `yum module` is used to install `wireshark` and finally the **state: latest** key-value pair is used to specify that the wireshark installed is the latest version.

The **second play** is dedicated to the **Ubuntu Server** (i.e. Load Balancer), it has two tasks: The **first task** is used to update the server. The `update_cahce` is similar to `apt update` and the **second task** is used to download the latest version of `wireshark` using the `apt module`.

### Step 9: Update GIT with the latest code

Now that all of your directories and files live on your local machine, you need to push changes made locally to GitHub. Remember you have been working on a separate branch `prj-145`, you need to get your branch peer-reviewed and pushed to the `main` branch. The following steps are taken to achieve this:

* Use the following commands to check the status of your branch, add files and directories then commit changes and push your branch to GitHub:

```sh
git status
```

```sh
git add inventory playbooks
```

```sh
git commit -m "commit message"
```

```sh
git push --set-upstream origin prj-145
```

* Got to your `ansible-config-mgt` repository on GitHub and click on the `Compare & pull request` button.

* Click on the `Create pull request` button.

* Click on the `Merge pull request` button.

* Click on the `Confirm merge` button.

* Head back to your terminal on VS Code, checkout from `prj-145` branch into the main and pull down the latest changes using the commands shown below:

```sh
git checkout main && git pull
```

* Once your code changes appear on the `main` branch, Jenkins will be triggered to do the `ansible` job and save all the files (i.e. build artifacts).


### Step 10: Run the first Ansible test

* SSH into the `Jenkins-Ansible` server.

```sh
ssh ubuntu@public_ip_address_of_jenkins_ansible
```

* The build artifacts are saved in the `/var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/` directory on the server.

```sh
cd /var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/ && ll
```

* Run the following command to run the Ansible Playbook.

```sh
ansible-playbook -i inventory/dev playbooks/common.yml
```

* Use the Ansible Adhoc command to check if wireshark has been installed on the servers.

```sh
ansible webservers -i inventory/dev -m command -a "wireshark --version"
```

```sh
ansible nfs,db -i inventory/dev -m command -a "wireshark --version"
```

```sh
ansible lb -i inventory/dev -m command -a "wireshark --version"
```

Your updated Ansible architecture now looks like this:

