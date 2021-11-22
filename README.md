# Automated ELK Stack Deployment

This document contains the following details:
- Description of the Topology
- ELK Configuration
- Access Policies
- Beats in Use
- Machines Being Monitored
- How to Use the Ansible Build


### Description of the Topology
This repository includes code defining the infrastructure below. 

![ELK Stack Deployment.png](https://github.com/zenithus/ELK-Stack-Project/blob/2531628f32c14c27ca5c99a5cd78aedd8b21b1ae/Diagrams/ELK%20Stack%20Deployment.PNG)

The main purpose of this network is to expose a load-balanced and monitored instance of DVWA, the "D*mn Vulnerable Web Application"

Load balancing ensures that the application will be highly **available**, in addition to restricting **inbound access** to the network. The load balancer ensures that work to process incoming traffic will be shared by the three vulnerable web servers. Access controls will ensure that only authorized users — namely, ourselves — will be able to connect in the first place.

Integrating an ELK server allows users to easily monitor the vulnerable VMs for changes to the **file systems of the VMs on the network**, as well as watch **system metrics**, such as CPU usage; attempted SSH logins; `sudo` escalation failures; etc.

The configuration details of each machine may be found below.

| Name         | Function    | IP Address | Operating System |
|--------------|-------------|------------|------------------|
| Jump Box     | Gateway     | 10.0.0.4   | Linux            |
| Web 1 (DVWA) | Web Server  | 10.0.0.5   | Linux            |
| WEb 2 (DVWA) | Web Server  | 10.0.0.6   | Linux            |
| Web 3 (DVWA) | Web Server  | 10.0.0.7   | Linux            | 
| ELK server   | Monitoring  | 10.1.0.4   | Linux            |

In addition to the above, Azure has provisioned a **load balancer** in front of all DVWA machines except for the jump box. The load balancer's targets are organized into the following availability zones:
- **Availability Zone 1**: Web 1 to 3 
- **Availability Zone 2**: ELK server

## ELK Server Configuration

The ELK VM exposes an Elastic Stack instance. **Docker** is used to download and manage an ELK container.

Rather than configure ELK manually, we opted to develop a reusable Ansible Playbook to accomplish the task. This playbook is duplicated below.


To use this playbook, one must log into the Jump Box, then issue: `ansible-playbook install_elk.yml`. This runs the `install_elk.yml` playbook on the `elk` host.

### Access Policies

The **jump box** machine can accept connections from the Internet. Access to this machine is only allowed from the IP address `115.70.22.154`which is the admin work station.

The **ELK server** can accept connections from the internet through port 5601 and the only IP address allowed is `115.70.22.154`which is the admin work station.

The machines on the internal network are _not_ exposed to the public Internet. Access to the internal network is via ssh on port 22 from the **jump box**.

The **ELK server** is also accessible via ssh on port 22 from the **jump box**. 

Machines _within_ the network can only be accessed by **each other**. The Web 1, Web 2 and Web 3 VMs send traffic to the ELK server.

A summary of the access policies in place can be found in the table below.

| Name           | Publicly Accessible | Allowed IP Addresses |
|----------------|---------------------|----------------------|
| Jump Box       | Yes                 | 115.70.22.154        |
| ELK            | Yes                 | 115.70.22.154        |
| Web 1 (DVWA)   | No                  | 10.0.0.1-254         |
| Web 2 (DVWA)   | No                  | 10.0.0.1-254         |
| Web 3 (DVWA)   | No                  | 10.0.0.1-254         |


### Elk Configuration

Ansible was used to automate configuration of the ELK machine. No configuration was performed manually, which is advantageous because It can easily deploy multi-tier applications. 
There is no need to configure applications on every machine, all the tasks are specified in the playbook. 

The command: 'ansible-playbook install_elk.yml' will automatically run the tasks in the playbook to each host machine through SSH.

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
#$ 
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

