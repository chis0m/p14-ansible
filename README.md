Dependences to be installed
====================================

```bash
sudo apt install python3 python3-pip wget unzip git -y
python3 -m pip install --upgrade setuptools
python3 -m pip install --upgrade pip
python3 -m pip install PyMySQL
python3 -m pip install mysql-connector-python
python3 -m pip install psycopg2==2.7.5 --ignore-installed
```

If you get the following error `error: subprocess-exited-with-error` then run
`sudo apt install libpq-dev python3-dev build-essential`


Ansible dependencies to install
=====================================

- For Mysql Database
ansible-galaxy collection install community.mysql
- For Postgresql Database
ansible-galaxy collection install community.postgresql


Import Setup to take note of
========================================
- Create `ansible.cfg` at `deploy/` and paste the following
```
[defaults]
timeout = 160
callback_whitelist = profile_tasks
log_path=~/ansible.log
host_key_checking = False
gathering = smart
ansible_python_interpreter=/usr/bin/python3
allow_world_readable_tmpfiles=true


[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=30m -o ControlPath=/tmp/ansible-ssh-%h-%p-%r -o ServerAliveInterval=60 -o ServerAliveCountMax=60 -o ForwardAgent=yes
```

- Export ansible config path in jenkins file
```jenkins
  environment {
    ANSIBLE_CONFIG="${WORKSPACE}/deploy/ansible.cfg"
  }
```

Important Setup to take note of In Jenkins Server
=============================
- Install Blue Ocean
- Generate Github Personal Access Token - Tick the following permissions `repo`, `read:user`, `user:email` 
- Use it to create a new pipeline for a github repo e.g https://github.com/chis0m/p14-ansible, so my project on jenkins will be p14-ansible 
- Goto `Dasbhoard -> p14-ansible -> Configure -> Build Configuration` change script path to `deploy/Jenkinsfile`
- SSH into the server and Install `ansible plugin` i.e `sudo apt install ansible`
- Goto `Global Configuration --> Ansible` and add ansible exec path e.g `/usr/bin/` for ubuntu
- On the browser, Goto `http://jenkins-ip:8080/credentials/store/system/domain/_/` click on Add `Credentials`
- Choose `SSH Username with private key`. Set ID: `ssh-private-key`,  username: `ubuntu`. CLick on `Enter directly` and add a copy of your ssh private key and save
- Goto `Dasbhoard -> p14-ansible -> pipeline-syntax` and generate playbook script.

 Set the following and leave every other thing as default

`Playbook file path in workspace: playbooks/site.yml`
`Inventory file path in workspace: inventory/{inventory}.yml`
`SSH connection credentials: ubunu`
`Use become: check`  
`Disable the host SSH key check: check`  
`Colorized output: check` 

- Copy the generated syntax and add to your Jenkinsfile
`ansiblePlaybook become: true, colorized: true, credentialsId: 'ssh-private-key', disableHostKeyChecking: true, installation: 'ansible', inventory: 'inventory/{inventory}.yml', playbook: 'playbooks/site.yml'`

NOTE: When using jenkins to deploy ansible, remove `ansible_ssh_private_key_file=~/.ssh/devum.pem` from inventory file because you have already added your ssh_key to jenkins

Install PHP Dependencies
========================
```bash
sudo apt install -y zip libapache2-mod-php phploc php-{xml,bcmath,bz2,intl,gd,mbstring,mysql,zip} mysql-client mysql
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
```

Install plugins for artifactory
===============================================
You need to install `Plot` and `Artifactory` plugins on Jenkins
- We will use plot plugin to display tests reports, and code coverage information.
- The Artifactory plugin will be used to easily upload code artifacts into an Artifactory server.

Setting up Jenkins with Artifactory server
==========================================
- Create and artifactory server
- It must be nothing less than 4gb of memory
- After creation, default username and password are `admin`, `password` respectively

#### Prerequisites
Install Artifactory on at least 2GM RAM, for AWS choose at least small or medium instance type.
Default ports 8081 (for Artifactory REST APIs) and 8082 (for UI, and all other product’s APIs) needs to be opened.

Configure Artifactory In Jenkins
=================================
- Goto Manage `Manage Jenkins --> Configure System`
- Scroll down to `JFrog`, the click `Add JFrog from platform instance`
- Instance ID `artifactory-server`, URL `http://<your-instance-public-address>:8082`
- username 'admin', password `your jfrog login password`
- Click on `Test Connection`
- Ignore the message
```
Found JFrog Artifactory 7.41.12 at http://54.160.148.114:8082/artifactory
JFrog Distribution not found at http://54.160.148.114:8082/distribution
```

Integrate Artifactory repository with Jenkins
==============================================
- Clone this repo `https://github.com/darey-devops/php-todo.git`
- Ensure the php packages have been installed as specified above
- Create a Jenkinsfile in the php-todo project
- Paste the following
```Jenkinsfile
pipeline {
    agent any

  stages {

     stage("Initial cleanup") {
          steps {
            dir("${WORKSPACE}") {
              deleteDir()
            }
          }
        }

    stage('Checkout SCM') {
      steps {
            git branch: 'main', url: 'https://github.com/chis0m/php-todo.git'
      }
    }

    stage('Prepare Dependencies') {
      steps {
             sh 'mv .env.sample .env'
             sh 'composer install'
             sh 'php artisan migrate'
             sh 'php artisan db:seed'
             sh 'php artisan key:generate'
      }
    }
   
    stage('Execute Unit Tests') {
      steps {
             sh './vendor/bin/phpunit'
      }
    }    
   
  }
}
```

- Goto your `mysSql roles` in p14-ansible project and new database `homestead`, user `homestead`, host `172.31.94.101` (Jenksins server Private IP)
- Update the database connectivity requirements in the file .env.sample of php-todo app
