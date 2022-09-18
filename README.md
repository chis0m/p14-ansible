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

Import Setup to take note of In Jenkins Server
=============================
- Install `ansible plugin`
- Goto `Global Configuration --> Ansible` and add ansible exec path e.g `/usr/bin/` for ubuntu
- Goto `http://ip:8080/credentials/store/system/domain/_/` click on Add `Credentials`
- Goto `pipeline-syntax` and generate playbook script

NOTE: When using jenkins to deploy ansible remove `ansible_ssh_private_key_file=~/.ssh/devum.pem` because you have already added your ssh_key to jenkins

Install PHP Dependencies
========================
```bash
sudo apt install -y zip libapache2-mod-php phploc php-{xml,bcmath,bz2,intl,gd,mbstring,mysql,zip}
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

#### Prerequisites
Install Artifactory on at least 2GM RAM, for AWS choose at least small or medium instance type.
Default ports 8081 (for Artifactory REST APIs) and 8082 (for UI, and all other product’s APIs) needs to be opened.
