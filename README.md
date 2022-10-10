Links:
[PHP TODO repo using laravel 9](https://github.com/chis0m/p14-php-todo)

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
- Install `ansible plugin` on Jenkins.
- SSH into the jenkins server and run `sudo apt install ansible -y`
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

- Copy the generated syntax and add to your Jenkinsfile add the appropriate step
`ansiblePlaybook become: true, colorized: true, credentialsId: 'ssh-private-key', disableHostKeyChecking: true, installation: 'ansible', inventory: 'inventory/{inventory}.yml', playbook: 'playbooks/site.yml'`

NOTE: When using jenkins to deploy ansible, remove `ansible_ssh_private_key_file=~/.ssh/devum.pem` from inventory file because you have already added your ssh_key to jenkins

Install PHP Dependencies (Jenkins Server)
========================
```bash
sudo apt install -y zip libapache2-mod-php phploc php-{xml,bcmath,bz2,intl,gd,mbstring,mysql,zip,curl} mysql-client
curl -sS https://getcomposer.org/installer | php && sudo mv composer.phar /usr/local/bin/composer
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
- Update the database connectivity requirements in the file .env.sample of php-todo app. The ip here should point to th DB


Add Code analysis
====================
In the jenkins file in php-todo, append the following:
```jenkinsfile
      steps {
            sh 'phploc app/ --log-csv build/logs/phploc.csv'

      }
    }

    stage('Plot Code Coverage Report') {
      steps {

            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Lines of Code (LOC),Comment Lines of Code (CLOC),Non-Comment Lines of Code (NCLOC),Logical Lines of Code (LLOC)                          ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'A - Lines of code', yaxis: 'Lines of Code'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Directories,Files,Namespaces', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'B - Structures Containers', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Average Class Length (LLOC),Average Method Length (LLOC),Average Function Length (LLOC)', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'C - Average Length', yaxis: 'Average Lines of Code'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Cyclomatic Complexity / Lines of Code,Cyclomatic Complexity / Number of Methods ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'D - Relative Cyclomatic Complexity', yaxis: 'Cyclomatic Complexity by Structure'      
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Classes,Abstract Classes,Concrete Classes', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'E - Types of Classes', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Methods,Non-Static Methods,Static Methods,Public Methods,Non-Public Methods', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'F - Types of Methods', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Constants,Global Constants,Class Constants', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'G - Types of Constants', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Test Classes,Test Methods', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'I - Testing', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Logical Lines of Code (LLOC),Classes Length (LLOC),Functions Length (LLOC),LLOC outside functions or classes ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'AB - Code Structure by Logical Lines of Code', yaxis: 'Logical Lines of Code'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Functions,Named Functions,Anonymous Functions', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'H - Types of Functions', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Interfaces,Traits,Classes,Methods,Functions,Constants', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'BB - Structure Objects', yaxis: 'Count'

      }
    }
```

Build Artifacts of the application
====================================
Append

Note: We have already created this `target` repository PBL in our artifcatory server

```jenkinsfile

    stage ('Package Artifact') {
      steps {
              sh 'zip -qr php-todo.zip ${WORKSPACE}/*'
      }
    }

    stage ('Upload Artifact to Artifactory') {
      steps {
        script { 
          def server = Artifactory.server 'artifactory-server'                 
          def uploadSpec = """{
            "files": [
              {
                "pattern": "php-todo.zip",
                "target": "PBL/php-todo",
                "props": "type=zip;status=ready"
                }
            ]
          }""" 

          server.upload spec: uploadSpec
        }
      }
    }
```


Deploy Application
====================
Append this:

```Jenkinsfile
    stage ('Deploy to Dev Environment') {
      steps {
        // changed feature/project14 to master 
        build job: 'p14-ansible/master', parameters: [[$class: 'StringParameterValue', name: 'env', value: 'dev']], propagate: false, wait: true
      }
    }
```

This will trigger the p14-ansible project with the dev inventory (build parameter)

- Go to p14-ansible project and add this to the playbook
```yaml
- name: Deploy the todo application
  import_playbook: ../static-assignments/deployment.yml
```
- Go to the deployment.yml and at task `Download the artifact`, update the artifactory details. Incase you dont know how to do that: 
  - go to artifactory server
  - navigate to the artificats and at the top, you would see `set me up`
  - Click on `set me up`, add your password
  - CLick on `Deploy`. Copy the credentials seen and update the deployment.yaml


  Setting Up SonarQube
  ======================
- Use EC2 ubuntu 2020
- Uncomment the sonarqube.yml and run script. Uses the `sonarqube` role
- visit <public-ip>:9000/sonar and login with username/password `admin` and `admin`
- Goto Jenkisns Server and install `SonarQube Scanner` plugin
- Goto `Manage Jenkins > Configure System`, look for **SonarQube Servers** and set name: `SonarQubeScanner` and server url: `http://<public-ip>:9000/`
- Generate authentication token in sonarqube. Goto `User > My Account > Security > Generate Tokens`
- Configure Quality Gate Jenkins Webhook in SonarQube – The URL should point to your Jenkins server http://{JENKINS_HOST}/sonarqube-webhook/. Goto `Administration > Configuration > Webhooks > Create`
- Update the Jenkinsfile in php-todo
  - Push to repository and run. **This would fail** but would have installed the scanner tool on the Linux server

```jenkinsfile
    stage('SonarQube Quality Gate') {
        environment {
            scannerHome = tool 'SonarQubeScanner'
        }
        steps {
            withSonarQubeEnv('sonarqube') {
                sh "${scannerHome}/bin/sonar-scanner"
            }

        }
    }
```
- Configure `sonar-scanner.properties`. SSH into Jenkins server and `cd /var/lib/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarQubeScanner/conf/
  ` and `sudo vi sonar-scanner.properties` and paste configuration related to php-todo
  
```text
sonar.host.url=http://<SonarQube-Server-IP-address>:9000
sonar.projectKey=php-todo
#----- Default source code encoding
sonar.sourceEncoding=UTF-8
sonar.php.exclusions=**/vendor/**
sonar.php.coverage.reportPaths=build/logs/clover.xml
sonar.php.tests.reportPath=build/logs/junit.xml
```

Conditionally deploy to higher environments
=============================================
- Update the Jenkinsfile of the php-todo to only to run Quality Gate whenever the running branch is either develop, hotfix, release, main, or master. Also add a time snippet so the pipeline would be complete only when sonarqube is successful
```jenkinsfile
    stage('SonarQube Quality Gate') {
      when { branch pattern: "^develop*|^hotfix*|^release*|^main*", comparator: "REGEXP"}
        environment {
            scannerHome = tool 'SonarQubeScanner'
        }
        steps {
            withSonarQubeEnv('sonarqube') {
                sh "${scannerHome}/bin/sonar-scanner -Dproject.settings=sonar-project.properties"
            }
            timeout(time: 1, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true
            }
        }
    }
```
  
