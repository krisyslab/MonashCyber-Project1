# Automated ELK Stack Deployment

This document contains the following details:
- Description of the Topology
- ELK Configuration
- Access Policies
- Beats in Use
- Machines Being Monitored
- Usage instructions


### Description of the Topology
This repository includes code defining the infrastructure below. 

![README Diagram.png](https://github.com/zenithus/ELK-Stack-Project/blob/ca272cf07f478a4658765454cc624cbbe3751bcd/Images/README%20Diagram.PNG)

The main purpose of this network is to expose a load-balanced and monitored instance of DVWA, the "D*mn Vulnerable Web Application"

Load balancing ensures that the application will be highly **available**, in addition to restricting **inbound access** to the network. The load balancer ensures that work to process incoming traffic will be shared by the three vulnerable web servers. Access controls will ensure that only authorized users — namely, ourselves — will be able to connect in the first place.

Integrating an ELK server allows users to easily monitor the vulnerable VMs for changes to the **file systems of the VMs on the network**, as well as watch **system metrics**, such as CPU usage; attempted SSH logins; `sudo` escalation failures; etc.

The configuration details of each machine may be found below.

| Name         | Function    | IP Address | Operating System | Specifications                                                   | Container                      |
|--------------|-------------|------------|------------------|------------------------------------------------------------------|--------------------------------|
| Jump Box     | Gateway     | 10.0.0.4   | Linux            | UbuntuServer, 18.4-lts-gen2, Standard_B1s, vCPUs 1, RAM 1GB      | cyberxsecurity/ansible: latest |
| Web 1 (DVWA) | Web Server  | 10.0.0.5   | Linux            | UbuntuServer, 18.4-lts-gen2, Standard_B1ms, vCPUs 1, RAM 2GB     | cyberxsecurity/dvwa            |
| WEb 2 (DVWA) | Web Server  | 10.0.0.6   | Linux            | UbuntuServer, 18.4-lts-gen2, Standard_B1ms, vCPUs 1, RAM 2GB     | cyberxsecurity/dvwa            |
| Web 3 (DVWA) | Web Server  | 10.0.0.7   | Linux            | UbuntuServer, 18.4-lts-gen2, Standard_B1ms, vCPUs 1, RAM 2GB     | cyberxsecurity/dvwa            |
| ELK server   | Monitoring  | 10.1.0.4   | Linux            | UbuntuServer, 18.4-lts--gen2, Standard_B2s, vCPUs 2, RAM 4GB     | sebp/elk:761                   |

In addition to the above, Azure has provisioned a **load balancer** in front of all DVWA machines except for the jump box. The load balancer's targets are organized into the following availability zones:
- **Availability Zone 1**: Web 1 to 3 
- **Availability Zone 2**: ELK server

### Steps in Setting up the Azure Cloud Network

1. Create Resource Group: web_zero resource group. When deciding which region to set up the infrastructure, choose the closest data centre the business operation or the users are to ensure low latency and high availability. This will be the region for the whole infrastructure.

2. Create the virtual network: web_zero_vnet.
The Virtual machines (VMs) are placed inside the virtual network. This is also where to define the network and subnet.

3. Set up the network security group: web_zero_nsg. The firewall has to be in place before creating the VMs. This will be responsible for protecting the virtual network by monitoring the traffic. Create an inbound rule of "Deny All" denying all traffic and give it a high priority number. example 4096 - the bigger the number the lesser is the priority.

4. Create the first virtual machine and add the user  - zeroAdmin.
JumpBoxProvisioner - this will be used to connect to the network later on. Instead of using a password to connect to the VM, create a secure connection through SSH. Open terminal or Gitbash - generate ssh-key and paste the id_rsa.pub into the security box while creating the VM. It should have web_zero_nsg as the security group.
Specifications: Ubuntu Server 18.04 with 1 CPU and 1 GB RAM. Give this VM a public IP. To connect run: ssh zeroAdmin@13.77.57.149

5. Create the VMs: Web-1, Web-2 and Web-3. Set up the subnet as default and without public IP. Use the same SSH key when connecting to them this time. Set the security group: web_zero_nsg
Specifications: 1 vCPU and 2 GB RAM Ubuntu 18.04 Standard - B1ms

Whiile creating the VMs make sure to choose the availability options and create a new Availability Set: web_zero_set. All VMs should belong to the same Availability set if they are to be placed in the same Load Balancer.

Create a new rule in the network security web_zero_nsg to allow SSH into the JumBoxProvisioner from the IP address of the workstation. Give a low priority number - example 100. Run the command: ssh zeroAdmin@13.77.57.149 to connect to the JumpBoxProvisioner

Install the docker: sudo apt install docker.io. Check the staus: sudo systemctl status docker. Pull the container: sudo docker pull cyberxsecurity/ansible. Then run the command: docker run -ti cyberxsecurity/ansible: latest bash. then exit. Create a new rule allowing SSH into the VMs with source IP 13.77.57.149 and destination: VirtualNetwork.

Open terminal or gitbash, connect to the container by running sudo docker run -it cyberxsecurity/ansible /bin/bash. Generate a new SSH key (id_rsa.pub) to replace all the public key that was initially used while creating the 3 VMs. 
Locate the Ansible config file and hosts file:  cd /etc/ansible. Run: nano /etc/ansible/hosts. unhash the [webserver] and add below it the IP addresses of the VMs and the location of the python script. example:
10.0.0.5 ansible_python_interpreter=/usr/bin/python3 then run: nano ansible.cfg look for the "remote_user" and change to remote_user = zeroAdmin. Try to ping the VMs: ansible -m ping all. The output should have "SUCCESS" to all 3 VMs.

Now deploy the configurations for the 3 VM servers using YAML. Create a YAML playbook file that will be used for the configurations. Run nano pentest.yml and edit it to add the configurations. after adding the configurations run the command ansible-playbook pentest.yml to deploy them to all the VMs. SSH to any of the VM servers to test. Run curl localhost/setup.php - it should output a DVWA html code which was included in the configuration for the next Cloud Security activity.

Create the Load balancer and give it a static Public IP address. Then create a health probe. Create a backend pool: web_zero_pool and add the 3 VMs. Add a Load Balance rule: web_zero_lbr that will forward port 80 traffic from the load balancer to the VirtualNetwork.

Create a new security rule in web_zero_nsg to allow port 80 traffic from the internet into the VirtualNetwork where the 3 VM servers are located.
Remove the "Deny All" rule that was set up earlier in step 3.
To test if they are accessible: go to a web browser and type -
http://20.190.121.162/setup.php

### Docker Container Setup for the **jump box**

Running the commands below will configure the **jump box** to run Docker containers and to install a container.

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
# Check for the installed container
zeroAdmin@JumpBoxProvisioner:~$ sudo docker container list -a
sudo docker start (container)
sudo docker ps
sudo docker attach (container)
**now check**
ssh username@Web-1IP
exit
ssh username@Web2-IP
exit
**now ping again**
ansible all -m ping
```

### Access Policies

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

### Elk Configuration

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

  ![docker_ps_output.png](https://github.com/zenithus/ELK-Stack-Project/blob/8006ca60d9860db93a77f3072e5e8b66b635e8a7/Images/docker_ps_output.png)


The playbook is duplicated below.

```yaml
---
  # install_elk.yml
  - name: Configure Elk VM with Docker
    hosts: elkservers
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

      # Use command module
      - name: Increase virtual memory
        command: sysctl -w vm.max_map_count=262144

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
```

### Target Machines & Beats
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
    # Use command module
    - name: Download filebeat .deb file
      command: curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.4.0-amd64.deb

    # Use command module
    - name: Install filebeat .deb
      command: dpkg -i filebeat-7.4.0-amd64.deb

    # Use copy module
    - name: Drop in filebeat config
      copy:
        src: /etc/ansible/files/filebeat-config.yml
        dest: /etc/filebeat/filebeat.yml

    # Use command module
    - name: Enable and Configure System Module
      command: filebeat modules enable system

    # Use command module
    - name: Setup filebeat
      command: filebeat setup

    # Use command module
    - name: Start filebeat service
      command: service filebeat start

    # Use systemd module
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
    # Use command module
  - name: Download metricbeat
    command: curl -L -O https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-7.4.0-amd64.deb

    # Use command module
  - name: install metricbeat
    command: dpkg -i metricbeat-7.4.0-amd64.deb

    # Use copy module
  - name: drop in metricbeat config
    copy:
      src: /etc/ansible/files/metricbeat-config.yml
      dest: /etc/metricbeat/metricbeat.yml

    # Use command module
  - name: enable and configure docker module for metric beat
    command: metricbeat modules enable docker

    # Use command module
  - name: setup metric beat
    command: metricbeat setup

    # Use command module
  - name: start metric beat
    command: service metricbeat start
```

### Using the Playbooks
In order to use the playbooks, you will need to have an Ansible control node already configured. We use the **jump box** for this purpose.


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

