# Automated ELK Stack Deployment

This document contains the following details:
- Description of the Topology
- Access Policies 
- Setting up the Azure Cloud Network
- Docker Container Setup for the **jump box**
- ELK Configuration
- Beats in Use
- Machines Being Monitored
- Usage instructions
- Advantages and Disadvantages of ELK stack
- Summary


## Description of the Topology
This repository includes code defining the infrastructure below. 

![README Diagram.png](https://github.com/krisyslab/ELK-Stack-Project/blob/6399ddeed4a590094b5d0297ef450ae39fd6693a/Images/README%20Diagram.PNG)

The main purpose of this network is to expose and observe a load-balanced and monitored instances of DVWA, the "D*mn Vulnerable Web Application"

Load balancing ensures that the application will be highly **available**, in addition to restricting **inbound access** to the network. The load balancer ensures that the processing of incoming traffic will be shared by the three vulnerable web servers. Access controls will ensure that only authorized users will be able to connect to the servers.

Integrating an ELK server allows users to easily monitor the vulnerable VMs.
Our focus is to monitor for changes to the following:
- **File systems of the VMs on the network**,  
- **System metrics such as CPU usage; attempted SSH logins and `sudo` escalation failures**
  
The configuration details of each machine may be found below.

| Name         | Function    | IP Address | Operating System | Specifications                                                   | Container                      |
|--------------|-------------|------------|------------------|------------------------------------------------------------------|--------------------------------|
| Jump Box     | Gateway     | 10.0.0.4   | Linux            | UbuntuServer, 18.4-lts-gen2, Standard_B1s, vCPUs 1, RAM 1GB      | cyberxsecurity/ansible: latest |
| Web 1 (DVWA) | Web Server  | 10.0.0.5   | Linux            | UbuntuServer, 18.4-lts-gen2, Standard_B1ms, vCPUs 1, RAM 2GB     | cyberxsecurity/dvwa            |
| WEb 2 (DVWA) | Web Server  | 10.0.0.6   | Linux            | UbuntuServer, 18.4-lts-gen2, Standard_B1ms, vCPUs 1, RAM 2GB     | cyberxsecurity/dvwa            |
| Web 3 (DVWA) | Web Server  | 10.0.0.7   | Linux            | UbuntuServer, 18.4-lts-gen2, Standard_B1ms, vCPUs 1, RAM 2GB     | cyberxsecurity/dvwa            |
| ELK server   | Monitoring  | 10.1.0.4   | Linux            | UbuntuServer, 18.4-lts--gen2, Standard_B2s, vCPUs 2, RAM 4GB     | sebp/elk:761                   |

In the above table, we have provisioned a **load balancer** in front of all three DVWA machines. The load balancer's targets are organized into the following availability zones:
- **Availability Zone 1**: Web 1(DVWA) to Web 3(DVWA)  
- **Availability Zone 2**: ELK server

The Jump box will not be placed in either of the Availability zones. 

## Access Policies

The **jump box** machine can accept connections from the Internet. Access to this machine is only allowed from the IP address of the admin work station using the current network security group rules.

The **ELK server** can accept connections from the internet through port 5601 and the only IP address allowed is that of the admin work station as well.

The machines on the internal network are _not_ exposed to the public Internet. Access to the internal network is via ssh on port 22 from the **jump box**.

The **ELK server** is also accessible via ssh on port 22 from the **jump box**. 

Machines _within_ the network can only be accessed by **each other**. The Web 1, Web 2 and Web 3 VMs send traffic to the ELK server.

A summary of the access policies in place can be found in the table below.

| Name           | Publicly Accessible | Open Ports                     | Allowed IP Addresses |
|----------------|---------------------|--------------------------------|----------------------|
| Jump Box       | Yes                 | Ports 22 and 80                | 115.70.22.154        |
| ELK            | Yes                 | Ports 22, 80, 5601, 9200, 5044 | 115.70.22.154        |
| Web 1 (DVWA)   | No                  | Port 22                        | 10.0.0.1-254         |
| Web 2 (DVWA)   | No                  | Port 22                        | 10.0.0.1-254         |
| Web 3 (DVWA)   | No                  | Port 22                        | 10.0.0.1-254         |

## Setting up the Azure Cloud Network

**These are based on the Network Topology as shown in the Diagram above** 

1. Create the `Resource Group`. A resource group is a collection of resources that share the same lifecycle, permissions, and policies. When deciding which region to set up the infrastructure, choose the closest data centre to the business operation or the location of most of the users to ensure low latency and high availability. This will be the region for the whole infrastructure.

2. Create the `Virtual Network`.
Virtual Networks in Azure are similar to a traditional network that you'd operate in your own data center, but brings with it additional benefits such as scale, availability, and isolation.The Virtual machines (VMs) are placed inside the virtual network. This is also where to define the network and subnet.

![create a virtual network.png](https://github.com/krisyslab/ELK-Stack-Project/blob/9e7db42bb2a5d4149a555409b86a05db2666f141/Images/create%20a%20virtual%20network.PNG)

3. Set up the `Network Security Group(NSG)`. Like the Virtual Network the NSG belongs to the Resource Group you created in step 1. This network security group will be selected when choosing the security group for the VMs of the newly created virtual network. The network security group rules must be configured before creating the VMs.

   Create an inbound rule of `Deny All` denying all inbound traffic and give it a high number e.g 4096 -- This rule will always be the last rule, so it should have the highest possible number for the priority. You should now have a VNet protected by a network security group that blocks all inbound traffic.

![deny all.png](https://github.com/krisyslab/ELK-Stack-Project/blob/9e7db42bb2a5d4149a555409b86a05db2666f141/Images/deny%20all.PNG)

Before you create the first Virtual Machine(VM), You first have to establish a secure way of accessing the VMs. Instead of using a password to connect to the VM, you create a secure connection through `SSH`. Open terminal or Gitbash on your workstation. 

   Generate the public key:

 ```bash
    dell@DESKTOP MINGW64 ~ ssh-keygen
    Generating public/private rsa key pair..........
    dell@DESKTOP MINGW64 ~
    $ cd .ssh
    dell@DESKTOP MINGW64 ~/.ssh
    $ ls
    id_rsa  id_rsa.pub  known_hosts 
    dell@DESKTOP MINGW64 ~/.ssh
    $ cat id_rsa.pub
    ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCpqBR/r3yyicrTPM+AZ4nQ3tQNeF25x/XhnGxetlBWM6yAWhllTkrP1+22WYxZ0MXZ6SoH82OXFmNqKBnJG5uA1mVW5Z979XDq80RxtBayNQMBGiwXSxduGfsLmkHRXy0mIOZogT9I2C+sRTeRZAPI4ssEfTFSsT6E0eM9pWF3jsSdIzRob10sDFzNcemwwhWEplq..........
 ```
    Highlight and copy the SSH key string to your clipboard. 

4. Create the `jump box VM`, your first virtual machines inside your cloud network, which is protected by your network security group. You will use this machine as a jump box to access your cloud network and any other machines inside your VNet. Give the jump box VM a `static Public IP address`.

![create JumpBox.png](https://github.com/krisyslab/ELK-Stack-Project/blob/9e7db42bb2a5d4149a555409b86a05db2666f141/Images/create%20JumpBox.PNG)

5. Create a new rule in the network security group to allow SSH into the jump box from the IP address of the workstation. Give a low number e.g 100 to ensure a higher priority. Check the IP address of your workstation - type `whatismyip` in google to see the IPV4. 

![allow ssh to jump box.png](https://github.com/krisyslab/ELK-Stack-Project/blob/9e7db42bb2a5d4149a555409b86a05db2666f141/Images/allow%20ssh%20to%20jump%20box.PNG)

6. Create the three VMs: Web-1, Web-2 and Web-3. In setting up each individual VM:
  - Set up the subnet as default and without public IP
  - Use the same SSH key that you provided for the jump box in step 5. This will be reset later on when you set up the containers.
  - Select the security group - the same security group selected for the jump box. 
  - In the Availability Options click create a new `Availability Set` name your availability set. All VMs should belong to the same Availability set if they are to be placed inside the same Load Balancer Backend Pool. 

![create VM.png](https://github.com/krisyslab/ELK-Stack-Project/blob/9e7db42bb2a5d4149a555409b86a05db2666f141/Images/create%20VM.PNG)

   Create a new rule allowing SSH into the VMs with source IP 10.0.0.4 (jump box VM) and destination: VirtualNetwork.

![allow ssh from JumpBox.png](https://github.com/krisyslab/ELK-Stack-Project/blob/9e7db42bb2a5d4149a555409b86a05db2666f141/Images/allow%20ssh%20from%20JumpBox.PNG)

7. Create the `Load balancer` and give it a `static Public IP address`. Create a `Health Probe`. Create a `Backend Pool` (Load Balancer Backend Pool) and add the 3 VMs (Web 1, Web 2, Web 3) in the backend pool.
![load balancer.png](https://github.com/krisyslab/ELK-Stack-Project/blob/cc03eb55c19f4d189f28f743359aaae475a356ac/Images/load%20balancer.PNG)
![backend pool.png](https://github.com/krisyslab/ELK-Stack-Project/blob/cc03eb55c19f4d189f28f743359aaae475a356ac/Images/backend%20pool.PNG)
![Health Probe.png](https://github.com/krisyslab/ELK-Stack-Project/blob/cc03eb55c19f4d189f28f743359aaae475a356ac/Images/Health%20Probe.PNG)
   Add a Load Balance rule that will forward port 80 traffic from the load balancer to the VirtualNetwork. A load balancing rule distributes incoming traffic that is sent to a selected IP address and port combination across a group of backend pool instances. Only backend instances that the health probe considers healthy receive new traffic.

![load balancing rule.png](https://github.com/krisyslab/ELK-Stack-Project/blob/9e7db42bb2a5d4149a555409b86a05db2666f141/Images/load%20balancing%20rule.PNG)   


## Docker Container Setup for the **jump box**

Running the commands below will configure the **jump box** to run Docker containers and to install a containers for the 3 VMs that you have created previously.

  ```bash
      # ssh to the jump box
      $ ssh zeroAdmin@13.77.57.149

      # install docker.io on your jump box
      zeroAdmin@JumpBoxProvisioner:~$ sudo apt update
      zeroAdmin@JumpBoxProvisioner:~$ sudo apt install docker.io

      # Verify that the Docker service is running
      zeroAdmin@JumpBoxProvisioner:~$ sudo systemctl status docker

      # If the Docker service is not running, start it with:
      zeroAdmin@JumpBoxProvisioner:~$ sudo systemctl start docker

      # Once Docker is installed, pull the container `cyberxsecurity/ansible`.
      zeroAdmin@JumpBoxProvisioner:~$ sudo docker pull cyberxsecurity/ansible

      # Launch the Ansible container
      zeroAdmin@JumpBoxProvisioner:~$ docker run -ti cyberxsecurity/ansible:latest bash

      # Check for the installed container. Remember the container name so that you know which to which one you have installed the container setup.
      zeroAdmin@JumpBoxProvisioner:~$ sudo docker container list -a
      #run the following commands to start and attach the container
      zeroAdmin@JumpBoxProvisioner:~$ sudo docker start (name of container)
      zeroAdmin@JumpBoxProvisioner:~$ sudo docker ps
      zeroAdmin@JumpBoxProvisioner:~$ sudo docker attach (name of container)
  ```
![docker container list.png](https://github.com/krisyslab/ELK-Stack-Project/blob/8c9b83d2f0b51527d6f9cbbfe77b817bfd94bcf7/Images/docker%20container%20list.PNG)

To reset your SSH key, you can do so in the VM details page by selecting 'Reset Password' on the left had column.
Generate a new SSH key (id_rsa.pub) from the JumpBoxProvisioner to replace all the public key that was initially used while creating the 3 VMs.

  ```bash
      root@9ba994bbeca9:~# ssh-keygen
      Generating public/private rsa key pair..........
      root@9ba994bbeca9:~# ls .ssh/
      id_rsa  id_rsa.pub
      root@9ba994bbeca9:~# cat .ssh/id_rsa.pub
      ssh-rsa ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDFg2PWQQ2hK+6lC+kdn5Lp+0H0FtSZfMtcv5d5IDtSw0rVP//66aoli2EUzi83P9oh2HCcTA4JdJOwy2yE1sJboi9XeuRsYDnUCpyqdIe8UnzwJEp1COHsFZBUcLEQ0XtN7Qv4tN8FHWU02P........
  ```
Go back to the Azure portal and `Reset the password` for the 3 DVWA VMs (Web 1, Web 2 and Web 3)

![reset VM password.png](https://github.com/krisyslab/ELK-Stack-Project/blob/9bac49833edc8a1ea2e0b7f70c1e793c9d0ba407/Images/reset%20VM%20password.PNG)   

This time go back to the terminal or Gitbash on your workstation and run these commands:

  ```bash
      # Locate the Ansible config file and hosts file
      root@9ba994bbeca9:~# cd /etc/ansible
      root@9ba994bbeca9:/etc/ansible# nano /etc/ansible/hosts

      # Uncomment the `[webservers]` header line and add below it the IP addresses of the VMs and the location of the python script and then save.
      # Add the internal IP address under the `[webservers]` header.
		  # Add the python line: `ansible_python_interpreter=/usr/bin/python3` besides each IP.
      `[webserver]
      10.0.0.5 ansible_python_interpreter=/usr/bin/python3
      10.0.0.6 ansible_python_interpreter=/usr/bin/python3
      10.0.0.7 ansible_python_interpreter=/usr/bin/python3`
      # Change the Ansible configuration file to use your admin account for SSH connections.
      root@9ba994bbeca9:/etc/ansible# nano ansible.cfg
      # Scroll down to the `remote_user` option. 
      # Uncomment the `remote_user` line and replace `root` with your admin username as shown below:
      for example `remote_user = <user-name-for-web-VMs>`
      # Test an Ansible connection using the appropriate Ansible command.
      root@9ba994bbeca9:/etc/ansible# ansible -m ping all
      10.0.0.6 | SUCCESS => {
          "changed": false,
          "ping": "pong"
      }
      10.0.0.5 | SUCCESS => {
          "changed": false,
          "ping": "pong"
      }
      10.0.0.7 | SUCCESS => {
          "changed": false,
          "ping": "pong"
      }
  ```
Now deploy the configurations for the 3 VM servers using YAML. Create a YAML playbook file that will be used for the configurations. Run `nano /etc/ansible/pentest.yml` and add the input below.

```yaml
---
  - name: Config Web VM with Docker
    hosts: webservers
    become: true
    tasks:
    # update_cache: yes is needed to download and install docker.io
      - name: docker.io
        apt:
          update_cache: yes
          name: docker.io
          state: present

    # Use the Ansible `pip` module to install `docker`
      - name: Install pip3
        apt:
          name: python3-pip
          state: present

    # installing the Python Docker Module, so Ansible can then utilize that module to control docker containers
      - name: Install Docker python module
        pip:
          name: docker
          state: present

    # Use the Ansible `docker-container` module to install the `cyberxsecurity/dvwa` container
      - name: download and launch a docker web container
        docker_container:
          name: dvwa
          image: cyberxsecurity/dvwa
          state: started
          restart_policy: always
          published_ports: 80:80

    # restart_policy: always` will ensure that the container restarts if you restart your web vm
      - name: Enable docker service
        systemd:
          name: docker
          enabled: yes
```
After saving the configurations run the command `ansible-playbook pentest.yml` to deploy them to all the VMs. 
You should be able to produce the same output as below when you run the playbook:

```bash

root@9ba994bbeca9:/etc/ansible# ansible-playbook pentest.yml
[WARNING]: ansible.utils.display.initialize_locale has not been called, this may result in incorrectly calculated
text widths that can cause Display to print incorrect line lengths

PLAY [Config Web VM with Docker] ***********************************************************************************
TASK [Gathering Facts] *********************************************************************************************
ok: [10.0.0.5]
ok: [10.0.0.6]
ok: [10.0.0.7]

TASK [docker.io] ***************************************************************************************************
changed: [10.0.0.5]
changed: [10.0.0.6]
changed: [10.0.0.7]

TASK [Install pip3] ************************************************************************************************
changed: [10.0.0.6]
changed: [10.0.0.7]
changed: [10.0.0.5]

TASK [Install Docker python module] ********************************************************************************
changed: [10.0.0.5]
changed: [10.0.0.6]
changed: [10.0.0.7]
  
TASK [download and launch a docker web container] ******************************************************************
[DEPRECATION WARNING]: The container_default_behavior option will change its default value from "compatibility"
to "no_defaults" in community.docker 2.0.0. To remove this warning, please specify an explicit value for it now. 
This feature will be removed from community.docker in version 2.0.0. Deprecation warnings can be disabled by 
setting deprecation_warnings=False in ansible.cfg.
changed: [10.0.0.5]
changed: [10.0.0.6]
changed: [10.0.0.7]

TASK [Enable docker service] ***************************************************************************************
changed: [10.0.0.6]
changed: [10.0.0.5]
changed: [10.0.0.7]

PLAY RECAP *********************************************************************************************************
10.0.0.5                   : ok=6    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
10.0.0.6                   : ok=6    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
10.0.0.7                   : ok=6    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```
Now try to ping all VM using the command `ansible -m ping all`
![ping all VM.png](https://github.com/krisyslab/ELK-Stack-Project/blob/aa71628c8ce68606e8650ea7475fe24cd4929efe/Images/ping%20all%20VM.PNG) 

SSH to any of the VM servers to test. Run `ssh [username]@[IP Address]`
![ssh to VM.png](https://github.com/krisyslab/ELK-Stack-Project/blob/63c901f38fe97cde9cd596e0eede6c75fb22d098/Images/ssh%20to%20VM.PNG)

Run `curl localhost/setup.php` - it should output a DVWA html code which was included in the configuration for the next Cloud Security activity.
![curl command on VM.png](https://github.com/krisyslab/ELK-Stack-Project/blob/63c901f38fe97cde9cd596e0eede6c75fb22d098/Images/curl%20command%20on%20VM.PNG)

Now go back to your Azure portal and create a new security rule in the network security group to allow port 80 traffic from the IP address of the workstation into the VirtualNetwork via the Public IP address of the Load Balancer. 

![allow http to VM.png](https://github.com/krisyslab/ELK-Stack-Project/blob/9e7db42bb2a5d4149a555409b86a05db2666f141/Images/allow%20http%20to%20VM.PNG)

Remove the "Deny All" rule that was set up earlier in step 3. Just remember that with the stated configuration, you will not be able to access these machines from another location unless the security Group rule is changed.

Verify that you can reach the DVWA app from your browser over the internet.

    - Open a web browser and enter the front-end IP address for your load balancer with `/setup.php` added to the IP address. In this example: http://20.190.121.162/setup.php

![DVWA website.png](https://github.com/krisyslab/ELK-Stack-Project/blob/0f2d5d755274c6f9e81292fb9eff9c0807b3bc1d/Images/DVWA%20website.PNG)


## ELK Server Configuration

The ELK VM exposes an Elastic Stack instance. **Docker** is used to download and manage an ELK container.

Rather than configure ELK manually, we opted to develop a reusable Ansible Playbook to accomplish the task. This playbook is duplicated below.

To use this playbook, one must log into the Jump Box, then issue: `ansible-playbook install_elk.yml`. This runs the `install_elk.yml` playbook on the `elk` host.

Ansible was used to automate configuration of the ELK machine. No configuration was performed manually, which is advantageous because It can easily deploy multi-tier applications. 
There is no need to configure applications on every machine, all the tasks are specified in the playbook. 

The command: 'ansible-playbook install_elk.yml elk' will automatically run the tasks in the playbook to each host machine through SSH.

The playbook implements the following tasks:

- Increase the memory before running the container. This is a system requirement for the ELK container. 
- Install docker.io, python3-pip and docker python module.
- After the docker is installed, it will download and run the `sebp/elk:761` container.
- The container will also run with these published ports: 5601:5601, 9200:9200 and 5044:5044
- Enable the service docker to run on boot.

The following screenshot displays the result of running `docker ps` after successfully configuring the ELK instance.

  ![docker_ps_output.png](https://github.com/krisyslab/ELK-Stack-Project/blob/9bac49833edc8a1ea2e0b7f70c1e793c9d0ba407/Images/docker_ps_output.png)


The playbook is duplicated below.

```yaml
---
  - name: Configure Elk VM with Docker
    hosts: elk
    remote_user: zeroAdmin
    become: true
    tasks:
      
      - name: Install docker.io
        apt:
          update_cache: yes
          name: docker.io
          state: present

      - name: Install pip3
        apt:
          force_apt_get: yes
          name: python3-pip
          state: present

      - name: Install Docker python module
        pip:
          name: docker
          state: present

      - name: Use more memory
        sysctl:
          name: vm.max_map_count
          value: "262144"
          state: present
          reload: yes

      - name: download and launch a docker elk container
        docker_container:
          name: elk
          image: sebp/elk:761
          state: started
          restart_policy: always
          published_ports:
            - 5601:5601
            - 9200:9200
            - 5044:5044

      - name: Enable service docker on boot
        systemd:
          name: docker
          enabled: yes
```

## Target Machines & Beats

This ELK server is configured to monitor the Web 1, Web 2 and Web 3 VMs, at `10.0.0.5`, `10.0.0.6` and `10.0.0.7`, respectively.

We have installed the following Beats on these machines:
- Filebeat
- Metricbeat

These Beats allow us to collect the following information from each machine:
- **Filebeat**: Filebeat detects changes to the filesystem. Specifically, we use it to collect Apache logs.
- **Metricbeat**: Metricbeat detects changes in system metrics, such as CPU usage. We use it to detect SSH login attempts, failed `sudo` escalations, and CPU/RAM statistics.

The playbook below installs Filebeat on the target hosts.

```yaml
---
  - name: Installing and Launch Filebeat
    hosts: webservers
    become: yes
    tasks:
    
    - name: Download filebeat .deb file
      command: curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.4.0-amd64.deb

    - name: Install filebeat .deb
      command: dpkg -i filebeat-7.4.0-amd64.deb

    - name: Drop in filebeat config
      copy:
        src: /etc/ansible/files/filebeat-config.yml
        dest: /etc/filebeat/filebeat.yml

    - name: Enable and Configure System Module
      command: filebeat modules enable system

    - name: Setup filebeat
      command: filebeat setup

    - name: Start filebeat service
      command: service filebeat start

    - name: Enable service filebeat on boot
      systemd:
        name: filebeat
        enabled: yes
```

The playbook below installs Metricbeat on the target hosts. 

```yaml
---
  - name: Install metric beat
    hosts: webservers
    become: true
    tasks:
  
    - name: Download metricbeat
      command: curl -L -O https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-7.4.0-amd64.deb

    - name: install metricbeat
      command: dpkg -i metricbeat-7.4.0-amd64.deb

    - name: drop in metricbeat config
      copy:
        src: /etc/ansible/files/metricbeat-config.yml
        dest: /etc/metricbeat/metricbeat.yml

    - name: enable and configure docker module for metric beat
      command: metricbeat modules enable docker

    - name: setup metric beat
      command: metricbeat setup

    - name: start metric beat
      command: service metricbeat start
```

## Usage Instructions

The ELK Stack is a collection of three open-source products —  `Elasticsearch `,  `Logstash `, and  `Kibana `. ELK stack provides centralized logging in order to identify problems with servers or applications. It allows you to search all the logs in a single place. It also helps to find issues in multiple servers by connecting logs during a specific time frame.

`E` stands for ElasticSearch: used for storing logs

`L` stands for LogStash : used for both shipping as well as processing and storing logs

`K` stands for Kibana: is a visualization tool (a web interface) which is hosted through Nginx or Apache

ElasticSearch, LogStash and Kibana are all developed, managed ,and maintained by the company named Elastic.

ELK Stack is designed to allow users to take data from any source, in any format, and to search, analyze, and visualize that data in real time.

In order to use the ELK Stack, you will need to have an Ansible control node already configured. Please refer to the `Docker Container Setup for the **jump box**` above.

To use the playbooks, we must perform the following steps:
- Copy the playbooks to the Ansible Control Node
- Run each playbook on the appropriate targets

The easiest way to copy the playbooks is to use Git:

```bash
$ cd /etc/ansible
$ mkdir files
# Clone Repository + IaC Files
$ git clone https://github.com/krisyslab/ELK-Stack-Project
# Move Playbooks and hosts file Into `/etc/ansible`
$ cp ELK-Stack-Project/Playbooks/* /etc/ansible
$ cp ELK-Stack-Project/Ansible/hosts /etc/ansible
```

This copies the playbook files to the correct place.

Next, you must create a `hosts` file to specify which VMs to run each playbook on. Run the commands below:

```bash
$ cd /etc/ansible
$ cat > hosts <<EOF
[webservers]
10.0.0.5 
10.0.0.6
10.0.0.7

[elk]
10.1.0.4
EOF
```

After this, the commands below run the playbook:

 ```bash
 $ cd /etc/ansible
 $ ansible-playbook install_elk.yml elk
 $ ansible-playbook install_filebeat.yml webservers
 $ ansible-playbook install_metricbeat.yml webservers
 ```

To verify success, wait five minutes to give ELK time to start up. 

Then, run: `curl http://10.1.0.4:5601/app/kibana`. This is the address of Kibana. If the installation succeeded, this command should print HTML to the console.

![curl command to kibana.png](https://github.com/krisyslab/ELK-Stack-Project/blob/79d5a42e1460d34f363d2735cc3d3672a96f566b/Images/curl%20command%20to%20kibana.PNG)

![data stream.png](https://github.com/krisyslab/ELK-Stack-Project/blob/79d5a42e1460d34f363d2735cc3d3672a96f566b/Images/data%20stream.PNG)

![file beat log.png](https://github.com/krisyslab/ELK-Stack-Project/blob/79d5a42e1460d34f363d2735cc3d3672a96f566b/Images/file%20beat%20log.PNG)

![metric.png](https://github.com/krisyslab/ELK-Stack-Project/blob/79d5a42e1460d34f363d2735cc3d3672a96f566b/Images/metric.PNG)

![sample kibana.png](https://github.com/krisyslab/ELK-Stack-Project/blob/79d5a42e1460d34f363d2735cc3d3672a96f566b/Images/sample%20kibana.PNG)
## Advantages and Disadvantages of ELK stack

### Advantages

ELK works best when logs from various Apps of an enterprise converge into a single ELK instance
It provides amazing insights for this single instance and also eliminates the need to log into hundred different log data sources
Rapid on-premise installation
Easy to deploy Scales vertically and horizontally
Elastic offers a host of language clients which includes Ruby. Python. PHP, Perl, .NET, Java, and JavaScript, and more
Availability of libraries for different programming and scripting languages

### Disadvantages

Different components In the stack can become difficult to handle when you move on to complex setup
There’s nothing like trial and error. Thus, the more you do, the more you learn along the way

### Summary

- Centralized logging can be useful when attempting to identify problems with servers or applications
- ELK server stack is useful to resolve issues related to centralized logging system
- ELK stack is a collection of three open source tools Elasticsearch, Logstash Kibana
- Elasticsearch is a NoSQL database
- Logstash is the data collection pipeline tool
- Kibana is a data visualization which completes the ELK stack
- In cloud-based environment infrastructures, performance and isolation is very important.