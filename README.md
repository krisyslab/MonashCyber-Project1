# Automated ELK Stack Deployment

This document contains the following details:
- Description of the Topology
- Setting up the Azure Cloud Network
- Docker Container Setup for the **jump box**
- Access Policies
- ELK Configuration
- Beats in Use
- Machines Being Monitored
- Usage instructions


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

      # Unhash the [webserver] and add below it the IP addresses of the VMs and the location of the python script and then save.
      `[webserver]
      10.0.0.5 ansible_python_interpreter=/usr/bin/python3
      10.0.0.6 ansible_python_interpreter=/usr/bin/python3
      10.0.0.7 ansible_python_interpreter=/usr/bin/python3`
      # Change the Ansible configuration file to use your admin account for SSH connections.
      root@9ba994bbeca9:/etc/ansible# nano ansible.cfg
      # Scroll down to the `remote_user` option. 
      # Uncomment the `remote_user` line and replace `root` with your admin username as shown below:
      for example `remote_user = zeroAdmin`
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
Now deploy the configurations for the 3 VM servers using YAML. Create a YAML playbook file that will be used for the configurations. Run `touch pentest.yml`

```yaml
---
  - name: Config Web VM with Docker
    hosts: webservers
    become: true
    tasks:
    
      - name: docker.io
        apt:
          update_cache: yes
          name: docker.io
          state: present

      - name: Install pip3
        apt:
          name: python3-pip
          state: present

      - name: Install Docker python module
        pip:
          name: docker
          state: present

      - name: download and launch a docker web container
        docker_container:
          name: dvwa
          image: cyberxsecurity/dvwa
          state: started
          restart_policy: always
          published_ports: 80:80

      - name: Enable docker service
        systemd:
          name: docker
          enabled: yes
```

![pentest.png](https://github.com/krisyslab/ELK-Stack-Project/blob/6e5255228aaf69fd813a239aaabd950c7653fcae/Images/pentest.PNG)

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



 and edit it to add the configurations. After adding the configurations run the command `ansible-playbook pentest.yml` to deploy them to all the VMs. SSH to any of the VM servers to test. `Run curl localhost/setup.php` - it should output a DVWA html code which was included in the configuration for the next Cloud Security activity.

8. Create a new security rule in network security group to allow port 80 traffic from the IP address of the workstation into the VirtualNetwork via the Public IP address of the Load Balancer. 

![allow http to VM.png](https://github.com/krisyslab/ELK-Stack-Project/blob/9e7db42bb2a5d4149a555409b86a05db2666f141/Images/allow%20http%20to%20VM.PNG)

   Remove the "Deny All" rule that was set up earlier in step 3. Just remember that with the stated configuration, you will not be able to access these machines from another location unless the security Group rule is changed.

   Verify that you can reach the DVWA app from your browser over the internet.

    - Open a web browser and enter the front-end IP address for your load balancer with `/setup.php` added to the IP address. In this example: http://20.190.121.162/setup.php

![DVWA website.png](https://github.com/krisyslab/ELK-Stack-Project/blob/0f2d5d755274c6f9e81292fb9eff9c0807b3bc1d/Images/DVWA%20website.PNG)

ssh username@Web-1IP
exit
ssh username@Web2-IP
exit
**now ping again**
ansible all -m ping

## Access Policies

The **jump box** machine can accept connections from the Internet. Access to this machine is only allowed from the IP address `115.70.22.154` which is the admin work station.

The **ELK server** can accept connections from the internet through port 5601 and the only IP address allowed is `115.70.22.154` which is the admin work station.

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
      # Use apt module
      - name: Install docker.io
        apt:
          update_cache: yes
          name: docker.io
          state: present

      # Use apt module
      - name: Install pip3
        apt:
          force_apt_get: yes
          name: python3-pip
          state: present

      # Use pip module
      - name: Install Docker python module
        pip:
          name: docker
          state: present

      # Use sysctl module
      - name: Use more memory
        sysctl:
          name: vm.max_map_count
          value: "262144"
          state: present
          reload: yes

      # Use docker_container module
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

      # Use systemd module
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

## Using the Playbooks
In order to use the playbooks, you will need to have an Ansible control node already configured. Please refer to the `Docker Container Setup for the **jump box**` above.

To use the playbooks, we must perform the following steps:
- Copy the playbooks to the Ansible Control Node
- Run each playbook on the appropriate targets

The easiest way to copy the playbooks is to use Git:

```bash
$ cd /etc/ansible
$ mkdir files
# Clone Repository + IaC Files
$ git clone https://github.com/zenithus/ELK-Stack-Project
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


---

## Complete Cloud Lab Walkthrough

In general, this document should not be needed. The following steps will guide you through stetting up the cloud lab resources in Azure in the event that students missed days or need to setup the lab all at once to use it for the upcoming project week.

These are the same steps that are covered throughout the activities in the cloud weeks.

**VERY IMPORTANT:** PAY ATTENTION TO THE USERS THAT YOU CREATE THROUGHOUT THE VARIOUS STEPS. THE EXERCISES BELOW PROVIDE EXAMPLE USERS, BUT MAKE SURE YOU NOTE AND USE THE USER THAT YOU CREATE.

---

#### Setting up the Resource Group

Use the Azure portal to create a resource group that will contain everything the Red Team needs in the cloud.

- On the home screen, search for "resource."

    ![](1/Images/resource_group/search_resource.png)

- Click on the **+ Create** button or the **Create resource group** button.

    ![](1/Images/ResourceGroupsInfo1.png)

- Create a name for your resource group and choose a region.        
    - Note: Choose a region that you can easily remember. Every resource you create after this must be created in the exact same region.

- Click on **Review + create**.

    ![](1/Images/resource_group/name_resource.png)

- Azure will alert you if there are any errors. Click on **Create** to finalize your settings and create the group.

    ![](1/Images/resource_group/create_resource.png)

- Once the group is created, click on **Go to resource group** in the top-right corner of the screen to view your new resource group.

    ![](1/Images/resource_group/go_to_resouce.png)

---

#### Setting up the VNet

Before you can deploy servers and services, there must be a network where these items can be accessed.

- This network should have the capacity to hold any resource that the Red Team needs, now and in the future.

- Return to the home screen and search for "net." Choose the search result for **Virtual networks**.

    ![](1/Images/virtual_net/search_network.png)

- Click on the **+ Create** button on the top-left of the page or the **Create virtual network** button on the bottom of the page.

    ![](1/Images/add_network1.png) 

Fill in the network settings:

- Subscription: Your free subscription should be the only option here.

- Resource group: This should be the resource group you created in step two.

- Name: A descriptive name so it will not get confused with other cloud networks in the same account.

- Region: Make sure to choose the same region you chose for your resource group. 

    - Carefully configuring the region of your resources is important for ensuring low latency and high availability. Resources should be located as close as possible to those who will be consuming them.

![](1/Activities/04_Virtual_Networking/Solved/Images/virtual_net/vNet1.png)

- IP Addresses: Azure requires you to define a network and subnet.
    - Use the defaults on this tab.

![](1/Activities/04_Virtual_Networking/Solved/Images/virtual_net/vNet2.png)

- Security: Leave the default settings.

![](1/Images/virtual_net/vNet3.png)

- Tags: No tags are needed.

![](1/Activities/04_Virtual_Networking/Solved/Images/virtual_net/vNet4.png)

Click **Create**.

![](1/Activities/04_Virtual_Networking/Solved/Images/virtual_net/create_network.png)

Once you have created your resource group and VNet, return to the home screen and choose the resource group option. 
- This provides a list of all resource groups in your account. 
- Choose the group that you created and you should see your VNet listed as a resource. 

![](1/Activities/04_Virtual_Networking/Solved/Images/virtual_net/final_resource_group.png)

You now have a resource group and VNet that you can use to create the rest of the cloud infrastructure throughout the unit.

---

#### Setting up a Network Security Group:

- On your Azure portal home screen, search "net" and choose **Network security groups**. 

![](1/Images/security_groups/search_net.png)

- Create a new security group.

- Add this security group to your resource group.

- Give the group a recognizable name that is easy to remember.

- Make sure the security group is in the same region that you chose during the previous activity.

![](1/Images/security_groups/create_nsg.png)

To create an inbound rule to block all traffic:

- Once the security group is created, click on the group to configure it.

- Choose **Inbound security rules** on the left.

- Click on the **+ Add** button to add a rule.

![](1/Images/security_groups/add_inbound_rule.png)

Configure the inbound rule as follows:

- Source: Choose **Any** source to block all traffic.

- Source port ranges: Source ports are always random, even with common services like HTTP. Therefore, keep the wildcard (*) to match all source ports.

- Destination: Select **Any** to block any and all traffic associated with this security group.

- Service: Select **Custom**

- Destination port ranges: Usually, you would specify a specific port or a range of ports for the destination. In this case, you can use the wildcard (*) to block all destination ports. You can also block all ports using a range like `0-65535`.

- Protocol: Block **Any** protocol that is used.

- Action: Use the **Block** action to stop all of the traffic that matches this rule.

- Priority: This rule will always be the last rule, so it should have the highest possible number for the priority. Other rules will always come before this rule. The highest number Azure allows is 4,096.

- Name: Give your rule a name like "Default-Deny."

![](1/Images/inbound_rule_settings1.png)

- Description: Write a quick description similar to "Deny all inbound traffic."

- Save the rule.

![](1/Images/security_groups/Overview.png)

You should now have a VNet protected by a network security group that blocks all traffic.

#### Setting up Virtual Machines

The goal of this activity was to set up your first virtual machines inside your cloud network, which is protected by your network security group. You will use this machine as a jump box to access your cloud network and any other machines inside your VNet.

---

Remember: Allowing a server to use password authentication for SSH is insecure because the password can be brute forced.

- Therefore, we will only use cryptographic SSH keys to access our cloud servers. Password authentication will not be allowed. 

- This is part of the "ground up" security approach that we have been discussing. 

Open your command line and run `ssh-keygen` to create a new SSH key pair.
    - DO NOT CREATE A PASSPHRASE, just press enter twice.

- Your output should be similar to:

    ```bash
    cyber@2Us-MacBook-Pro ~ % ssh-keygen
    Generating public/private rsa key pair.
    Enter file in which to save the key (/Users/cyber/.ssh/id_rsa):
    Enter passphrase (empty for no passphrase): 
    Enter same passphrase again: 
    Your identification has been saved in id_rsa.
    Your public key has been saved in id_rsa.pub.
    The key fingerprint is:
    SHA256:r3aBFU50/5iQbbzhqXY+fOIfivRFdMFt37AvLJifC/0 cyber@2Us-MacBook-Pro.local
    The randomart image is:
    +---[RSA 2048]----+
    |         .. . ...|
    |          o. =..+|
    |         o .o *=+|
    |          o  +oB+|
    |        So o .*o.|
    |        ..+...+ .|
    |          o+++.+ |
    |        ..oo=+* o|
    |       ... ..=E=.|
    +----[SHA256]-----+
    ```

Run `cat ~/.ssh/id_rsa.pub` to display your `id_rsa.pub` key:

- Your output should be similar to:

    ```bash
    cyber@2Us-MacBook-Pro ~ % cat ~/.ssh/id_rsa.pub 

    ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDGG6dBJ6ibhgM09U+kn/5NE7cGc4CNHWXein0f+MciKElDalf76nVgFvJQEIImMhAGrtRRJDAd6itlPyBpurSyNOByU6LX7Gl6DfGQKzQns6+n9BheiVLLY9dtodp8oAXdVEGles5EslflPrTrjijVZa9lxGe34DtrjijExWM6hBb0KvwlkU4worPblINx+ghDv+3pdrkUXMsQAht/fLdtP/EBwgSXKYCu/
    ```

- Highlight and copy the SSH key string to your clipboard. 

---

#### VM 1 - Jump-Box

Open your Azure portal and search for "virtual machines."

- Use the **+ Add** button or the **Create virtual machine** button to create a new VM.

    ![](1/Images/VM/CreateVM.png)

Use the following settings for this VM: 

- Resource group: Choose the same resource group that you created for the Red Team.

- Virtual machine name: Use the name "Jump Box Provisioner."

- Region: Use the same region that you used for your other resources.
	- Note that availability of VM's in Azure could cause you to change the region where your VM's are created.
	- The goal is to create 3 machines in the same resource group attached to the same security group. If you cannot add 3 machines to the resource group and security group that you have, a new resource group and security group may need to be created in another region.

- Availability options: We will use this setting for other machines. For our jump box, we will leave this on the default setting.

- Image: Choose the Ubuntu Server 18.04 option.

- Choose the VM option that has:
  - Whose offering is **Standard - B1s**
  - 1 CPU
  - 1 RAM

For SSH, use the following settings: 

- Authentication type: SSH public key.

- Username: Create any username you like.

- SSH public key: Paste the public key string that you copied earlier.

- Public inbound ports: Ignore this setting. It will be overwritten when you choose your security group.

- Select inbound ports: Ignore this setting. It will be overwritten when you choose your security group.

![](1/Images/VM/VMSettings.png)

Move to the **Networking** tab and set the following settings:

- Virtual network: Choose the VNet you created for the Red Team.

- Subnet: Choose the subnet that you created earlier.

- Public IP: This can be kept as default. 

- NIC network security group: Choose the Advanced option so we can specify our custom security group.

- Configure network security group: Choose your Red Team network security group.

- Accelerated networking: Keep as the default setting (Off).

- In the Networking settings, take note of the VM URL. You may use it later.

- Load balancing: Keep as the default setting (No).

    ![](1/Images/VM/VMNetworking.png)

- Click on **Review + create**.

    ![](1/Images/VM/FinalizeVM.png)

- Finalize all your settings and create the VM by clicking on the **Create** button.

#### VM's 2 and 3 - Web VM's

Create 2 more new VMs. Keep the following in mind when configuring these VM's:
- Each VM should be named "Web-1" and "Web-2"

- These VM's need to be in the same resource group you are using for all other resources.

- The VM's should be located in the same region as your resource group and security group.
	- Note that availability of VM's in Azure could cause you to change the region where your VM's are created.
	- The goal is to create 3 machines in the same resource group attached to the same security group. If you cannot add 3 machines to the resource group and security group that you have, a new resource group and security group may need to be created in another region.

- The administrative username should make sense for this scenario. You should use the same admin name for all 3 machines. Make sure to take a note of this name as you will need it to login later.

- **SSH Key:** Later in the lab setup, we will overwrite these SSH keys. For now, use the SSH key that you created in the first VM setup.
    - Run: `cat ~/.ssh/id_rsa.pub` and copy the key.

- Choose the VM option that has:
  - Whose offering is **Standard - B1ms**
  - 1 CPU
  - 2 RAM

**Note:** These web machines should have _2 GB_ of RAM and the Jump-Box only needs _1 GB_. All 3 machines should only have _1 vCPU_ because the free Azure account only allows _4 vCPU's_ in total per region.

**VERY IMPORTANT:** Make sure both of these VM's are in the same availability Set. Machines that are not in the same availability set cannot be added to the same load balancer later, and will have to be deleted and recreated in the same availability set.  
- Under Availability Options, select 'Availability Set'. Click on 'Create New' under the Availability set. Give it an appropriate name. After creating it on the first VM, choose it for the second VM.

![](1/Images/Avail_Set/Avail-Set.png)

In the **Networking** tab and set the following settings:

- Virtual network: Choose the VNet you created for the Red Team.

- Subnet: Choose the subnet that you created earlier.

- Public IP: NONE! Make sure these web VM's do not have a public IP address.

![](1/Images/Avail_Set/No-Ip.png)

- NIC network security group: Choose the Advanced option so we can specify our custom security group.

- Configure network security group: Choose your Red Team network security group.

- Accelerated networking: Keep as the default setting (Off).

- In the Networking settings, take note of the VM URL. You may use it later.

- Load balancing: Keep as the default setting (No).

**NOTE:** Notice that these machines will not be accessible at this time because our security group is blocking all traffic. We will configure access to these machines in a later activity.

The final WebVM's should resemble the following:

![](1/Images/Avail_Set/final-VM.png)

#### Setting up your Jump Box Administration

The goal of this activity was to create a security group rule to allow SSH connections only from your current IP address, and to connect to your new virtual machine for management.

---

1. Visit `whatsmyip.org` to get the IPv4 address of the network you are currently using.


Next, log into portal.azure.com to create a security group rule to allow SSH connections from your current IP address.

2. Find your security group listed under your resource group.

3. Create a rule allowing SSH connections from your IP address. 

    - Choose **Inbound security rules** on the left.

    - Click **+ Add** to add a rule.

        - Source: Use the **IP Addresses** setting, with your IP address in the field.

        - Source port ranges: Set to **Any** or '*' here.
            - Destination: This can be set **VirtualNetwork** but a better setting is to specify the internal IP of your jump box to really limit this traffic.

        - Service: This can be set to **SSH**

        - Destination port ranges: Since we chose SSH, it will default to `22`.

        - Protocol: This will default to **TCP**.

        - Action: Set to **Allow** traffic.

        - Priority: This must be a lower number than your rule to deny all traffic, i.e., less than 4,096. 

        - Name: Name this rule anything you like, but it should describe the rule. For example: `SSH`.

        - Description: Write a short description similar to: "Allow SSH from my IP." 


	![](2/Images/limit-ip1.png)


4. Use your command line to SSH to the VM for administration. Windows users should use GitBash.

    - The command to connect is `ssh admin-username@VM-public-IP`.

    - Use the username you previously set up. (Your SSH key should be used automatically.)

5. Once you are connected, check your `sudo` permissions.
    
    - Run the command `sudo -l`.

    - Notice that your admin user has full `sudo` permissions without requiring a password.

Please note that your public IP address will change depending on your location. 

-  In a normal work environment, you would set up a static IP address to avoid continually creating rules to allow access to your cloud machine. 

 - In our case, you will need to create another security rule allowing your home network to access your Azure VM. 


**NOTE:** If you need to reset your SSH key, you can do so in the VM details page by selecting 'Reset Password' on the left had column.

![](2/Images/SSH-Jump/password-reset.png)

---

#### Docker Container Setup

The goal of this activity was to configure your jump box to run Docker containers and to install a container.

1. Start by installing `docker.io` on your Jump box.

    - Run `sudo apt update` then `sudo apt install docker.io`
    
  ![](2/Images/Docker_Ansible/Docker_Install.png)

2. Verify that the Docker service is running.

    - Run `sudo systemctl status docker`

      - **Note:** If the Docker service is not running, start it with `sudo systemctl start docker`.

 ![](2/Images/Docker_Ansible/Docker_Process.png)

3. Once Docker is installed, pull the container `cyberxsecurity/ansible`.

    - Run `sudo docker pull cyberxsecurity/ansible`.

![](2/Images/Docker_Ansible/Docker_Pull.png)

    - You can also switch to the root user so you don't have to keep typing `sudo`.

    - Run `sudo su`.


4. Launch the Ansible container and connect to it using the appropriate Docker commands.

    - Run `docker run -ti cyberxsecurity/ansible:latest bash` to start the container.

    - Run `exit` to quit.

![](2/Images/Docker_Ansible/Container_Connected.png)

5. Create a new security group rule that allows your jump box machine full access to your VNet.

    - Get the private IP address of your jump box.

    ![](2/Images/Docker_Ansible/VM_IP_Address.png)

    - Go to your security group settings and create an inbound rule. Create rules allowing SSH connections from your IP address.

       - Source: Use the **IP Addresses** setting with your jump box's internal IP address in the field.

        - Source port ranges: **Any** or * can be listed here.

        - Destination: Set to **VirtualNetwork**.

        - Service: Select **SSH**

        - Destination port ranges: This will default to port `22`.

        - Protocol: This will default to **TCP**.

        - Action: Set to **Allow** traffic from your jump box.

        - Priority: Priority must be a lower number than your rule to deny all traffic.

        - Name: Name this rule anything you like, but it should describe the rule. For example: `SSH from Jump Box`.

        - Description: Write a short description similar to: "Allow SSH from the jump box IP."

        ![](2/Images/JumpBox_settings1.png)

Your final security group rules should be similar to this:

![](2/Images/Docker_Ansible/Security_Rules.png)

---

#### Setup your Provisioner

In this activity, you launched a new VM from the Azure portal that could only be accessed using a new SSH key from the container running inside your jump box.

1. Connect to your Ansible container. Once you're connected, create a new SSH key and copy the public key.

    - Run `docker images` to view your image.

    - Run `docker run -it cyberxsecurity/ansible /bin/bash` to start your container and connect to it. (Note that the prompt changes.)

        ```bash
        root@Red-Team-Web-VM-1:/home/RedAdmin# docker run -it cyberxsecurity/ansible /bin/bash
        root@23b86e1d62ad:~# 
        ```

    - Run `ssh-keygen` to create an SSH key.

        ```bash
        root@23b86e1d62ad:~# ssh-keygen
        Generating public/private rsa key pair.
        Enter file in which to save the key (/root/.ssh/id_rsa): 
        Created directory '/root/.ssh'.
        Enter passphrase (empty for no passphrase): 
        Enter same passphrase again: 
        Your identification has been saved in /root/.ssh/id_rsa.
        Your public key has been saved in /root/.ssh/id_rsa.pub.
        The key fingerprint is:
        SHA256:gzoKliTqbxvTFhrNU7ZwUHEx7xAA7MBPS2Wq3HdJ6rw root@23b86e1d62ad
        The key's randomart image is:
        +---[RSA 2048]----+
        |  . .o+*o=.      |
        |   o ++ . +      |
        |    *o.+ o .     |
        |  . =+=.+ +      |
        |.. + *.+So .     |
        |+ . +.* ..       |
        |oo +oo o         |
        |o. o+.  .        |
        | .+o.  E         |
        +----[SHA256]-----+
        root@23b86e1d62ad:~#
        ```

    - Run `ls .ssh/` to view your keys.

        ```bash
        root@23b86e1d62ad:~# ls .ssh/
        id_rsa  id_rsa.pub
        ```

    - Run `cat .ssh/id_rsa.pub` to display your public key.

        ```bash
        root@23b86e1d62ad:~# cat .ssh/id_rsa.pub 
        ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDz5KX3urPPKbYRKS3J06wyw5Xj4eZRQTcg6u2LpnSsXwPWYBpCdF5lE3tJlbp7AsnXlXpq2G0oAy5dcLJX2anpfaEBTEvZ0mFBS24AdNnF3ptan5SmEM/
        ```

    - Copy your public key string.

2. Return to the Azure portal and locate one of your web-vm's details page.

		- Reset your Vm's password and use your container's new public key for the SSH user.

    - Get the internal IP for your new VM from the Details page.

![](3/Images/web-reset-ssh/reset-ssh.png)

3. After your VM launches, test your connection using `ssh` from your jump box Ansible container. 
    - Note: If only TCP connections are enabled for SSH in your security group rule, ICMP packets will not be allowed, so you will not be able to use `ping`. 

    ```bash
    root@23b86e1d62ad:~# ping 10.0.0.6
    PING 10.0.0.6 (10.0.0.6) 56(84) bytes of data.
    ^C
    --- 10.0.0.6 ping statistics ---
    4 packets transmitted, 0 received, 100% packet loss, time 3062ms

    root@23b86e1d62ad:~#
    ```

    ```bash
    root@23b86e1d62ad:~# ssh [username]@[IP Address]
    The authenticity of host '10.0.0.6 (10.0.0.6)' can't be established.
    ECDSA key fingerprint is SHA256:7Wd1cStyhq5HihBf+7TQgjIQe2uHP6arx2qZ1YrPAP4.
    Are you sure you want to continue connecting (yes/no)? yes
    Warning: Permanently added '10.0.0.6' (ECDSA) to the list of known hosts.
    Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 5.0.0-1027-azure x86_64)

    * Documentation:  https://help.ubuntu.com
    * Management:     https://landscape.canonical.com
    * Support:        https://ubuntu.com/advantage

    System information as of Mon Jan  6 18:49:56 UTC 2020

    System load:  0.01              Processes:           108
    Usage of /:   4.1% of 28.90GB   Users logged in:     0
    Memory usage: 36%               IP address for eth0: 10.0.0.6
    Swap usage:   0%


    0 packages can be updated.
    0 updates are security updates.


    Last login: Mon Jan  6 18:33:30 2020 from 10.0.0.4
    To run a command as administrator (user "root"), use "sudo <command>".
    See "man sudo_root" for details.

    ansible@Pentest-1:~$
    ```

    - Exit this SSH session by running `exit`.

4. Locate the Ansible config file and hosts file.

    ```bash
    root@1f08425a2967:~# ls /etc/ansible/
    ansible.cfg  hosts  roles
    ```
     - Add this machine's internal IP address to the Ansible hosts file.

    - Open the file with `nano /etc/ansible/hosts`.
    - Uncomment the `[webservers]` header line.
    - Add the internal IP address under the `[webservers]` header.
		- Add the python line: `ansible_python_interpreter=/usr/bin/python3` besides each IP.

    ```bash
        # This is the default ansible 'hosts' file.
        #
        # It should live in /etc/ansible/hosts
        #
        #   - Comments begin with the '#' character
        #   - Blank lines are ignored
        #   - Groups of hosts are delimited by [header] elements
        #   - You can enter hostnames or ip addresses
        #   - A hostname/ip can be a member of multiple groups
        # Ex 1: Ungrouped hosts, specify before any group headers.

        ## green.example.com
        ## blue.example.com
        ## 192.168.100.1
        ## 192.168.100.10

        # Ex 2: A collection of hosts belonging to the 'webservers' group

        [webservers]
        ## alpha.example.org
        ## beta.example.org
        ## 192.168.1.100
        ## 192.168.1.110
        10.0.0.6 ansible_python_interpreter=/usr/bin/python3
				10.0.0.7 ansible_python_interpreter=/usr/bin/python3
        ```

5. Change the Ansible configuration file to use your administrator account for SSH connections.

    - Open the file with `nano /etc/ansible/ansible.cfg` and scroll down to the `remote_user` option.
    
    - Uncomment the `remote_user` line and replace `root` with your admin username using this format:
				- `remote_user = <user-name-for-web-VMs>`

	Example:
    ```bash
    # What flags to pass to sudo
    # WARNING: leaving out the defaults might create unexpected behaviours
    #sudo_flags = -H -S -n

    # SSH timeout
    #timeout = 10

    # default user to use for playbooks if user is not specified
    # (/usr/bin/ansible will use current user as default)
    remote_user = sysadmin

    # logging is off by default unless this path is defined
    # if so defined, consider logrotate
    #log_path = /var/log/ansible.log

    # default module name for /usr/bin/ansible
    #module_name = command

    ```

6. Test an Ansible connection using the appropriate Ansible command.

If you used `ansible_python_interpreter=/usr/bin/python3` your output should look like:

```bash
10.0.0.5 | SUCCESS => {
"changed": false, 
"ping": "pong"
}
10.0.0.6 | SUCCESS => {
		"changed": false, 
		"ping": "pong"
}
```

If that line isn't present, you will get a warning like this:

```bash
root@1f08425a2967:~# ansible -m ping all
[DEPRECATION WARNING]: Distribution Ubuntu 18.04 on host 10.0.0.6 should use 
/usr/bin/python3, but is using /usr/bin/python for backward compatibility with 
prior Ansible releases. A future Ansible release will default to using the 
discovered platform python for this host. See https://docs.ansible.com/ansible/
2.9/reference_appendices/interpreter_discovery.html for more information. This 
feature will be removed in version 2.12. Deprecation warnings can be disabled 
by setting deprecation_warnings=False in ansible.cfg.
10.0.0.6 | SUCCESS => {
		"ansible_facts": {
				"discovered_interpreter_python": "/usr/bin/python"
		}, 
		"changed": false, 
		"ping": "pong"
}
```

- Ignore the `[DEPRECATION WARNING]` or add the line `ansible_python_interpreter=/usr/bin/python3` next to each Ip address in the hosts file.

---

#### Setup your Ansible Playbooks

The goal here is to create an Ansible playbook that installed Docker and configure a VM with the DVWA web app.

---

1. Connect to your jump box, and connect to the Ansible container in the box. 

    - If you stopped your container or exited it in the last activity, find it again using `docker container list -a`.

    ```bash
    root@Red-Team-Web-VM-1:/home/RedAdmin# docker container list -a
    CONTAINER ID        IMAGE                           COMMAND                  CREATED             STATUS                         PORTS               NAMES
    Exited (0) 2 minutes ago                           hardcore_brown
    a0d78be636f7        cyberxsecurity/ansible:latest   "bash"                   3 days ago  
    ```
    - **NOTE:** In this example, the container is called `hardcore_brown` and this name is randomly generated. The name of your container will be different.

   - Start the container again using `docker start [container_name]`.

    ```bash
    root@Red-Team-Web-VM-1:/home/RedAdmin# docker start hardcore_brown
    hardcore_brown
    ```

   - Get a shell in your container using `docker attach [container_name]`.

    ```bash
    root@Red-Team-Web-VM-1:/home/RedAdmin# docker attach hardcore_brown
    root@1f08425a2967:~#
    ```

2. Create a YAML playbook file that you will use for your configuration. 

  ```bash
  root@1f08425a2967:~# nano /etc/ansible/pentest.yml
  ```

   The top of your YAML file should read similar to:

```YAML
---
- name: Config Web VM with Docker
    hosts: web
    become: true
    tasks:
```

- Use the Ansible `apt` module to install `docker.io` and `python3-pip`:
**Note:** `update_cache` must be used here, or `docker.io` will not install. (this is the equivalent of running `apt update`)

  ```YAML
    - name: docker.io
      apt:
				update_cache: yes
        name: docker.io
        state: present

    - name: Install pip3
      apt:
        force_apt_get: yes
        name: python3-pip
        state: present
  ```

Note: `update_cache: yes` is needed to download and install docker.io

- Use the Ansible `pip` module to install `docker`:

  ```bash
    - name: Install Python Docker module
      pip:
        name: docker
        state: present
  ```

Note: Here we are installing the Python Docker Module, so Ansible can then utilize that module to control docker containers. More about the Python Docker Module [HERE](https://docker-py.readthedocs.io/en/stable/)

- Use the Ansible `docker-container` module to install the `cyberxsecurity/dvwa` container.
  - Make sure you publish port `80` on the container to port `80` on the host.
  ```YAML
    - name: download and launch a docker web container
      docker_container:
        name: dvwa
        image: cyberxsecurity/dvwa
        state: started
        restart_policy: always
        published_ports: 80:80
  ```

NOTE: `restart_policy: always` will ensure that the container restarts if you restart your web vm. Without it, you will have to restart your container when you restart the machine.

You will also need to use the `systemd` module to restart the docker service when the machine reboots. That block looks like this:

```YAML
    - name: Enable docker service
      systemd:
        name: docker
        enabled: yes
```

3. Run your Ansible playbook on the new virtual machine.

    Your final playbook should read similar to:
    ```YAML
    ---
    - name: Config Web VM with Docker
      hosts: webservers
      become: true
      tasks:
      - name: docker.io
        apt:
          force_apt_get: yes
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

      - name: download and launch a docker web container
        docker_container:
          name: dvwa
          image: cyberxsecurity/dvwa
          state: started
          published_ports: 80:80

      - name: Enable docker service
        systemd:
          name: docker
          enabled: yes
    ```

  - Running your playbook should produce an output similar to the following:

    ```bash
    root@1f08425a2967:~# ansible-playbook /etc/ansible/pentest.yml

    PLAY [Config Web VM with Docker] ***************************************************************

    TASK [Gathering Facts] *************************************************************************
    ok: [10.0.0.6]

    TASK [docker.io] *******************************************************************************
    [WARNING]: Updating cache and auto-installing missing dependency: python-apt

    changed: [10.0.0.6]

    TASK [Install pip3] *****************************************************************************
    changed: [10.0.0.6]

    TASK [Install Docker python module] ************************************************************
    changed: [10.0.0.6]

    TASK [download and launch a docker web container] **********************************************
    changed: [10.0.0.6]

    PLAY RECAP *************************************************************************************
    10.0.0.6                   : ok=5    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
    ```

4. To test that DVWA is running on the new VM, SSH to the new VM from your Ansible container.

    - SSH to your container:

    ```bash
    root@1f08425a2967:~# ssh sysadmin@10.0.0.6
    Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 5.0.0-1027-azure x86_64)

    * Documentation:  https://help.ubuntu.com
    * Management:     https://landscape.canonical.com
    * Support:        https://ubuntu.com/advantage

      System information as of Mon Jan  6 20:01:03 UTC 2020

      System load:  0.01              Processes:              122
      Usage of /:   9.9% of 28.90GB   Users logged in:        0
      Memory usage: 58%               IP address for eth0:    10.0.0.6
      Swap usage:   0%                IP address for docker0: 172.17.0.1


    18 packages can be updated.
    0 updates are security updates.


    Last login: Mon Jan  6 19:33:51 2020 from 10.0.0.4
    ```

    - Run `curl localhost/setup.php` to test the connection. If everything is working, you should get back some HTML from the DVWA container.

    ```bash
    ansible@Pentest-1:~$ curl localhost/setup.php

    <!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">

    <html xmlns="http://www.w3.org/1999/xhtml">

      <head>
        <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />

        <title>Setup :: Damn Vulnerable Web Application (DVWA) v1.10 *Development*</title>

        <link rel="stylesheet" type="text/css" href="dvwa/css/main.css" />

        <link rel="icon" type="\image/ico" href="favicon.ico" />

        <script type="text/javascript" src="dvwa/js/dvwaPage.js"></script>

      </head>
    ```

---

#### Setting up the Load Balancer

To complete this activity, you had to install a load balancer in front of the VM to distribute the traffic among more than one VM.

---

1. Create a new load balancer and assign it a static IP address.

    - Start from the homepage and search for "load balancer."
![](3/Images/Load-Balancer/LBSearch.png)

- Click **+ Create** to create a new load balancer.
    - It should have a static public IP address. 
    - Click **Create** to create the load balancer.
![](3/Images/CreateLB1.png)
![](3/Images/Load-Balancer/FinalizeLB.png)


2. Add a health probe to regularly check all the VMs and make sure they are able to receive traffic.
![](3/Images/Load-Balancer/HealthProbe.png)

3. Create a backend pool and add BOTH of your VM's to it.
![](3/Images/PoolSettings1.png)

---

#### Setting up the NSG to expose port 80

 To complete this activity, you had to configure the load balancer and security group to work together to expose port `80` of the VM to the internet.

1. Create a load balancing rule to forward port `80` from the load balancer to your Red Team VNet.

    - Name: Give the rule an appropriate name that you will recognize later.

    - IP Version: This should stay on **IPv4**.

    - Frontend IP address: There should only be one option here.

    - Protocol: Protocol is **TCP** for standard website traffic.

    - Port: Port is `80`.

    - Backend port: Backend port is also `80`.

    - Backend pool and Health probe: Select your backend pool and your health probe.

    - Session persistence: This should be changed to **Client IP and protocol**.
        - Remember, these servers will be used by the Red Team to practice attacking machines. If the session changes to another server in the middle of their attack, it could stop them from successfully completing their training.

    - Idle timeout: This can remain the default (**4 minutes**).

    - Floating IP: This can remain the default (**Disabled**).

    ![](3/Images/Load-Balancer/LBRuleSettings.png)


2. Create a new security group rule to allow port `80` traffic from the internet to your internal VNet.

    - Source: Change this your external IPv4 address.

    - Source port ranges: We want to allow **Any** source port, because they are chosen at random by the source computer.

    - Destination: We want the traffic to reach our **VirtualNetwork**.

    - Service: Select **HTTP**

    - Destination port ranges: This will default to port `80`, as you just selected HTTP

    - Protocol: This will default to **TCP** 

    - Action: Set to **Allow** traffic.

    - Name: Choose an appropriate name that you can recognize later.

![](3/Images/HTTP-SG/HTTP-Rule.png)

3. Remove the security group rule that blocks _all_ traffic on your vnet to allow traffic from your load balancer through.

    - Remember that when we created this rule we were blocking traffic from the allow rules that were already in place. One of those rules allows traffic from load balancers.

    - Removing your default deny all rule will allow traffic through.

4. Verify that you can reach the DVWA app from your browser over the internet.

    - Open a web browser and enter the front-end IP address for your load balancer with `/setup.php` added to the IP address.

        - For example: `http://40.122.71.120/setup.php`

![](3/Images/HTTP-SG/DVWA-Test.png)

**Note:** With the stated configuration, you will not be able to access these machines from another location unless the security Group rule is changed.

---
#### END

Congratulations! You have created a highly available web server for XCorp's Red Team to use for testing and training.

---
© 2020 Trilogy Education Services, a 2U, Inc. brand. All Rights Reserved.

