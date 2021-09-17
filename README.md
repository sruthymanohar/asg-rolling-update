# ASG rolling update using Jenkins and Ansible
[![Build Status](https://travis-ci.org/joemccann/dillinger.svg?branch=master)](https://travis-ci.org/joemccann/dillinger)

Here is a  project  for rolling autoscale group using jenkins and ansible. The main concern of this project is when developers push the  new versions of website to github, it should  automattically reflected to the website.

On the infra side, an  autoscaling group is created with  launch configuration and it conatins user data for fetching datas from github.  And it's connected to load balancer for balancing the traffic.

For automating the whole project I have used jenkin server with webhook feature in github. When a developer push new data to github, the webhook configuration triggers the jenkin. After that,the ansible plugin in jenkin run the ansible playbook automattically and the new changes will be added to  instances in autoscaling group. Here at a time one instance will be updated so we can avoid the downtime of client applications. 

Architecture:

![alt text](https://github.com/sruthymanohar/asg-rolling-update/blob/main/architecture.png)

Features:
- Continuous Deployment with Jenkins, playbook automattically run using  ansible plugin in jenkins.
- When a  a new version of site contents are uploaded in github it automattically pushes to    main website.
- Dynamic Inventory Is configured
- At a time one instance will be updated so we can avoid downtime of client applications
   
    
 Pre-Requests :
 - Basic knowledge in aws services, ansible, jenkins, git
 -  IAM role with necessary privilleges or you can use awscli to configure IAM user on ansible  master server.
 -  A master ansible server. You can install ansible on master server using follwing url 
  
    https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html

Modules Used: 

    yum
    pip
    ec2_instance_info
    add_hosts
    seiral
    file
    git
    pause
    debug
    ansible-vault

Ansible Playbook:

```sh 
---
- name: "Creating Dynamic Invetory Of Autoscaling Group"
  hosts: localhost
  vars:
    - variable.vars
  tasks:
    - name: "Installing pip"
      yum:
        name: pip
        state: present

    - name: "Installing boto3"
      pip:
        name: boto3
        state: present

    - name: "Autoscale - Getting Ec2 Instacne details"
      ec2_instance_info:
        region: "{{ region }}"
        filters:
          "tag:aws:autoscaling:groupName": "{{ tag_name }}"
          instance-state-name: [ "running"]
      register: asg_instances


    - name: "Autoscale - Creating Inventory Of Autoscaling Ec2"
      add_host:
        name: "{{ item.public_ip_address }}"
        groups: "asg"
        ansible_host: "{{ item.public_ip_address }}"
        ansible_port: 22
        ansible_user: "ec2-user"
        ansible_ssh_private_key_file: "{{ private_key }}"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
      with_items:
        - "{{ asg_instances.instances }}"


- name: "Deploying Website From Github To Asg"
  become: true
  hosts: asg
  serial: 1
  vars_files:
    - variables.vars
  tasks:

    - name: "Installing Packages"
      yum:
        name:
          - httpd
          - git
        state: present

    - name: "Crating Clone Directory"
      file:
        path: "{{ cloneDir }}"
        state: directory

    - name: "Cloning GitRepository"
      git:
        repo: "{{ gitRepository }}"
        dest: "{{ cloneDir }}"
      register: cloneStatus

    - name: "Disabling HealthCheck"
      when: cloneStatus.changed
      file:
        path: "/var/www/html/health.txt"
        state: file
        mode: 0000

    - name: "Off Loading Ec2 From ElB"
      when: cloneStatus.changed
      pause:
        seconds: "{{ health_time }}"

    - name: "Moving Contents From CloneDir To DocumentRoot"
      when: cloneStatus.changed
      copy:
        src: "{{ cloneDir }}"
        dest: /var/www/html/
        remote_src: true
    - name: "Moving contents to index.html"
      replace:
        path: /var/www/html/index.html
        regexp: 'hostname'
        replace: '{{ ansible_fqdn }}'


    - name: "Restarting/Enabling httpd"
      when: cloneStatus.changed
      service:
        name: httpd
        state: restarted
        enabled: true

    - name: "Disabling HealthCheck"
      when: cloneStatus.changed
      file:
        path: "/var/www/html/health.txt"
        state: file
        mode: 0644

    - name: "Loading Ec2 Back To ElB"
      when: cloneStatus.changed
      pause:
        seconds: "{{ health_time }}"
```


Here variables are initialized in a separate file named variable.vars.

```sh 
region: "Enter your region name"
tag_name: "Mention-ASG-tag-Name"
private_key: "Key-file"
gitRepository: "Mention-Your-Git-Repository url"
clonDir: Mention-Required-Clone-Directory
```

The userdata used for launch configuration is pasted below:

```sh 
#!/bin/bash
yum install httpd php git -y
git clone https://github.com/sruthymanohar/ansible_website.git /var/git/
cp -pr /var/git/* /var/www/html/
service httpd restart
chkconfig httpd on
```

### Jenkin Server Setup

For jenkin installation  you can use the following command
```
#amazon-linux-extras install epel -y
#yum install java-1.8.0-openjdk-devel -y
#wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
#rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
#yum -y install jenkins
#systemctl start jenkins
#systemctl enable jenkins
```

You can access the jenkin using following url.

http://65.2.151.139:8080/

The default port of jenkin is 8080.

1. The initial step is installation of ansible module. 
   ```sh
   Manage Jenkins --> Manage Plugins
   ```
   ![alt text](https://github.com/sruthymanohar/asg-rolling-update/blob/main/Capture1.PNG)
   

2. The next step is configure the ansible plugin via Global Tool Configuration
   ![alt text](https://github.com/sruthymanohar/asg-rolling-update/blob/main/image2.PNG)
   
3. Then updates with the name and executable path for the ansible plugin.
  ![alt text](https://github.com/sruthymanohar/asg-rolling-update/blob/main/picture3.PNG)
  
4. Next create a free style project  
   
   Dashboard -->New item
   
   Then update github repository url as shown in the image.
   
   ![alt text](https://github.com/sruthymanohar/asg-rolling-update/blob/main/Capture5.PNG)
   
   Then apply build procedure "Invoke Ansible Playbook" and ansible playbook location
   
   ![alt text](https://github.com/sruthymanohar/asg-rolling-update/blob/main/Capture6.PNG)
 
   We can provide additional security for the ansible file with ansible-vault. Once the jenikn is configured update Github webhook with jenkin server details.
   
   ![alt text](https://github.com/sruthymanohar/asg-rolling-update/blob/main/Capture7.PNG)
   
   Once the github is configured return to Jenkins configuration as such to trigger when GitHub pushes
   
   ![alt text](https://github.com/sruthymanohar/asg-rolling-update/blob/main/Capture8.PNG)
   
   # conclusion
   
 Here is the rolling automated autoscale group. The automation is done with jenkin and webhook feature in git.
