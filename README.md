# Kubernetes Multi-Node Cluster Configuration using Ansible

![k8s_multi_node_cluster](https://miro.medium.com/max/1124/1*C68oi2HkYy4k-tfIL-5_Ig.jpeg)

## Content
- **Project Understanding : Dynamic Inventory Setup**
- **Project Understanding : Ansible Roles Setup**

## Project Understanding : Dynamic Inventory Setup
**What is Dynamic Inventory for AWS in Ansible ?**
<br>
_**Dynamic inventory** is an **Ansible** plugin that makes an API call to **AWS** to get the instance information in the run time. It gives the **ec2** instance details **dynamically** to manage the **AWS** infrastructure._
<br><br>
Lets understand the procedure to setup Dynamic Inventory for AWS EC2 Instance one by one :
- Create a directory for installing **dynamic inventory** modules for Amazon EC2.

```shell
mkdir dynamic_inventory
```

- Install dynamic inventory modules i.e., ec2.py and ec2.ini, also both files are interdependent on each other. **ec2.ini** is responsible for storing information related to AWS account whereas **ec2.py** is responsible for executing modules that collects the information of instances launched on AWS.

```
Command for creation of ec2.py dynamic inventory file 👇

wget   https://raw.githubusercontent.com/ansible/ansible/stable-2.9/contrib/inventory/ec2.py

Command for creation of ec2.ini dynamic inventory file 👇

wget   https://raw.githubusercontent.com/ansible/ansible/stable-2.9/contrib/inventory/ec2.ini
```

<br>

_**Note**_: Install **wget** command in case it’s not present. The command for the same(for RHEL) is mentioned below:

```shell
sudo yum install wget
```

![wget](https://miro.medium.com/max/875/1*rwYZ-ei92hWF2ULv5sCHTA.png)

- Next, make the dynamic inventory files executable using chmod command.

```shell
chmod +x ec2.py

chmod +x ec2.ini
```

![chmod](https://miro.medium.com/max/718/1*vZDQXhqGmK9UCQ1iYhGVkw.png)

- Key needs to be provided in order to login to the instances newly launched. Also key with **.pem** format works in this case and not the one with **.ppk** format. Permission needs to be provided to respective key to set it up in **read mode**.

```shell
chmod 400 keypair.pem
```

![chmod400](https://miro.medium.com/max/555/1*7ZpQik5O4mm6WuqKBgnpcQ.png)

_**Note**_:

```
Permissions in Linux

0 -> No Permission      -> ---
1 -> Execute            -> --x
2 -> Write              -> -w-
3 -> Execute+Write      -> -wx
4 -> Read               -> r--
5 -> Read+Execute       -> r-x
6 -> Read+Write         -> rw-
7 -> Read+Write+Execute -> rwx
```

**Ansible Configuration File Setup**
  1. Mention the path to the directory created for installing dynamic inventory module under inventory keyword in the configuration file.
  2. Since Ansible works on SSH protocol for Linux OS and it prompts yes/no by default when used, in order to disable it , host_key_checking needs to be set to false.
  3. Warnings given by commmands could be disabled by setting command_warnings to false.
  4. The path to the private key could be provided under private_key_file in the configuration file.


![config_file](https://miro.medium.com/max/875/1*U91ZWZkhQcV6xOYLYcHU2g.png)

```
[defaults]

inventory = path_to_directory
host_key_checking = false
command_warnings = false
private_key_file = path_to_key
ask_pass = false

[privilege_escalation]
become=true
become_method=sudo
become_user=root
become_ask_pass=false
```

- Let’s execute the command to check if there is any issue in the dynamic inventory files in the directory containing the same.

![ansible_list_host](https://miro.medium.com/max/875/1*UrqbFTgKRWX2k8HhX0kIRQ.png)

- As per the output correct above, some changes needs to be made in the respective file. In case of ec2.py, version specified needs to be changed i.e., from **python** to **python3**.

![ec2py](https://miro.medium.com/max/875/1*yf8YnRIhvxC12I-6TJvWaA.png)

<p align="center"><b>ec2.py</b></p><br><br>

```shell
#!/usr/bin/python3
```

- Also, we need to update regions, AWS account **access_key** and **secret_access_key** inside ec2.ini file.

![ec2ini](https://miro.medium.com/max/875/1*zyOeluG0hWEjSvR48kaxPw.png)

<p align="center"><b>ec2.ini</b></p><br><br>

![ec2iniregion](https://miro.medium.com/max/875/1*c8_jgM7ismxNkwO29CcdCg.png)

<p align="center"><b>Region specified is ‘ap-south-1’ : ec2.ini</b></p><br><br>

- After the changes made in the previous step, export **aws_region**, **aws_access_key** and **aws_access_secret_key** on the command line.

```shell
export AWS_REGION='ap-south-1'(in this case)
export AWS_ACCESS_KEY=XXXX
export AWS_ACCESS_SECRET_KEY=XXXX
```

- After the following setup, the command used before provides the list of hosts in AWS properly. It indicates that the particular issue has been resolved successfully.

![ansiblelisthosts](https://miro.medium.com/max/875/1*FruMvZLDfm9YRNFS-q_wkg.png)

<br>
<p align="center"><b>. . .</b></p><br>

## Project Understanding : Ansible Roles Setup
Let’s us understand the implementation part by part

### Part 1 : Ansible Playbook (main.yml)

```yaml
- hosts: localhost
  gather_facts: no
  roles:
  - role: aws_ec2_setup
  
- hosts: tag_Name_k8smaster
  remote_user: ec2-user
  roles:
  - role: k8s_master
  
- hosts: tag_Name_k8sworker
  gather_facts: no
  remote_user: ec2-user
  roles:
  - role: k8s_worker
```

- **aws_ec2_setup** : Role for AWS setup
- **k8s_master** : Role for Master Node setup in k8s cluster
- **k8s_worker** : Role for Worker Node setup in k8s cluster

— — — — — — — — — — — — — — — — — — — — — — — — —

### Part 2 : Ansible Role (aws_ec2_setup)

#### <ins>tasks : main.yml</ins>

- _**“Installation of boto3 library”**_: Installs **boto3** library(Python3 SDK for AWS)

```yaml
- name: Installation of boto3 library
  pip:
    name: boto3
    state: present
```

- _**“Creation of VPC for EC2 Instance”**_: Creates **AWS VPC(Virtual Private Cloud)** using the **ec2_vpc_net** module. The required parameters to be specified are **cidr_block** and **name**.

```yaml
- name: Creation of VPC for EC2 Instance
  ec2_vpc_net:
    aws_access_key: "{{ aws_access_key }}"
    aws_secret_key: "{{ aws_secret_key }}"
    region: "{{ region }}"
    name: "{{ vpc_name}}"
    cidr_block: "{{ cidr_block_vpc }}"
    state: present
  register: ec2_vpc
```

![vpc](https://miro.medium.com/max/875/1*Ja4aVfOCC8OjMmkonY2XMw.png)

- _**“Internet Gateway for VPC”**_: Sets up internet gateway for VPC created in the previous task.

```yaml
- name: Internet Gateway for VPC
  ec2_vpc_igw:
    aws_access_key: "{{ aws_access_key }}"
    aws_secret_key: "{{ aws_secret_key }}"
    region: "{{ region }}"
    state: present
    vpc_id: "{{ ec2_vpc.vpc.id  }}"
    tags:
      Name: "{{ igw_name }}"
  register: igw_info
```

![igw](https://miro.medium.com/max/875/1*e3OIBMsuxOOGATJwu1Kgzw.png)

- _**“VPC Subnet Creation”**_: Creates Subnet for VPC created before in the specified availability zone in the region. **map_public** should be set to **yes** so that the instances launched within it would be assigned public IP by default.

```yaml
- name: VPC Subnet Creation
  ec2_vpc_subnet:
    aws_access_key: "{{ aws_access_key }}"
    aws_secret_key: "{{ aws_secret_key }}"
    region: "{{ region }}"
    vpc_id: "{{ ec2_vpc.vpc.id  }}"
    az: "{{ availability_zone }}"
    state: present
    cidr: "{{ cidr_subnet  }}"
    map_public: yes
    tags:
      Name: "{{ subnet_name }}"
  register: subnet_info
```

![subnet](https://miro.medium.com/max/875/1*vs4zeJUl3WCyVwruKSDH3w.png)

- _**“Creation of VPC Subnet Route Table”**_: Creates Route Table and associates it with the subnet. Also, it specifies the route to the internet gateway created before in the route table.

```yaml
- name: Creation of VPC Subnet Route Table
  ec2_vpc_route_table:
    aws_access_key: "{{ aws_access_key }}"
    aws_secret_key: "{{ aws_secret_key }}"
    region: "{{ region }}"
    vpc_id: "{{ ec2_vpc.vpc.id  }}"
    state: present
    tags:
      Name: "{{ route_table_name }}"
    subnets: [ "{{ subnet_info.subnet.id }}" ]
    routes:
      - dest: 0.0.0.0/0
        gateway_id: "{{ igw_info.gateway_id }}"
```

![subnet_route_table](https://miro.medium.com/max/875/1*SdXb1hhYMY68nMNzw8DhTQ.png)

- _**“Security Group Creation for Master Node”**_: Creates Security Group for **Master Node** of Kubernetes Cluster.

```yaml
- name: Security Group Creation for Master Node
  ec2_group:
    aws_access_key: "{{ aws_access_key }}"
    aws_secret_key: "{{ aws_secret_key }}"
    region: "{{ region }}"
    name: "{{ sg_name_master }}"
    vpc_id: "{{ ec2_vpc.vpc.id  }}"
    state: present
    description: Security Group for Master Node
    tags:
      Name: "{{ sg_name_master }}"
    rules:
    - proto: tcp
      ports:
        - 8090
        - 10250
        - 22
        - 10255
        - 6443
      cidr_ip: 0.0.0.0/0
    - proto: udp
      ports:
        - 8472
      cidr_ip: 0.0.0.0/0
  register: security_group_master
```

![security_group_master_node](https://miro.medium.com/max/875/1*uW91vBrELAR969CJf1JxBw.png)

- _**“Security Group Creation for Worker Node”**_: Creates Security Group for **Worker Node** of Kubernetes Cluster.

```yaml
- name: Security Group Creation for Worker Node
  ec2_group:
    aws_access_key: "{{ aws_access_key }}"
    aws_secret_key: "{{ aws_secret_key }}"
    region: "{{ region }}"
    name: "{{ sg_name_worker }}"
    vpc_id: "{{ ec2_vpc.vpc.id  }}"
    state: present
    description: Security Group for Worker Node
    tags:
      Name: "{{ sg_name_worker }}"
    rules:
    - proto: tcp
      ports:
        - 8090
        - 10250
        - 22
        - 10255
      cidr_ip: 0.0.0.0/0
    - proto: udp
      ports:
        - 8472
      cidr_ip: 0.0.0.0/0
  register: security_group_worker
```

![security_group_worker_node](https://miro.medium.com/max/875/1*wVVMkTHYqS19itRoOtfXEA.png)

- _**“Creation of EC2 Instance for Master Node”**_: Creates EC2 Instance for Kubernetes Cluster **Master Node**.

```yaml
- name: Creation of EC2 Instance for Master Node
  ec2:
    aws_access_key: "{{ aws_access_key }}"
    aws_secret_key: "{{ aws_secret_key }}"
    region: "{{ region }}"
    image: "{{ image_id }}"
    exact_count: 1
    instance_type: "{{ instance_type }}"
    vpc_subnet_id: "{{ subnet_info.subnet.id }}"
    key_name: "{{ key_name }}"
    group_id: "{{ security_group_master.group_id }}"
    wait: yes
    wait_timeout: 600
    instance_tags:
      Name: k8smaster
    count_tag:
      Name: k8smaster
```

![ec2_instance_master_node](https://miro.medium.com/max/875/1*OwIdxdU0T5mZGfyeMXDcXg.png)

- _**“Creation of EC2 Instance for Worker Node”**_: Creates EC2 Instance for Kubernetes Cluster **Worker Node**.

```yaml
- name: Creation of EC2 Instance for Worker Node
  ec2:
    aws_access_key: "{{ aws_access_key }}"
    aws_secret_key: "{{ aws_secret_key }}"
    region: "{{ region }}"
    image: "{{ image_id }}"
    instance_type: "{{ instance_type }}"
    vpc_subnet_id: "{{ subnet_info.subnet.id }}"
    key_name: "{{ key_name }}"
    group_id: "{{ security_group_worker.group_id }}"
    exact_count: "{{ instance_count }}"
    wait: yes
    wait_timeout: 600
    instance_tags:
      Name: k8sworker
    count_tag:
      Name: k8sworker
```

![ec2_instance_worker_node](https://miro.medium.com/max/875/1*p-O-SfjduFpb87NB4JtbeQ.png)

- _**“meta: refresh_inventory”**_: Meta tasks are a special kind of task that influences Ansible internal execution or state. **refresh_inventory** is the meta task that reloads the dynamic inventory generated. In this case, it reloads the dynamic inventory that makes it easier to scale up or scale down the number of EC2 Instances without much hindrance.

```yaml
- meta: refresh_inventory
```

- _**“pause”**_: This module is used for pausing Ansible playbook execution. In this case, it pauses the execution to provide meta task to refresh the dynamic inventory.

```yaml
- pause:
      minutes: 4
```

_**Note**_:

1. The main reason behind specifying the **instance_tags** for both Master Node and Worker Node, as dynamic inventory would create host groups accordingly which provides a proper distinction between instances associated to both Master and Worker Node. The list of dynamic host groups could be obtained using the command mentioned below:

```shell
./ec2.py --list
```

![instance_tags](https://miro.medium.com/max/500/1*98S2edtKOOS_d7z3QSXphQ.png)

2. Parameters i.e., **aws_access_key**, **aws_secret_key** and **region** specified in each task above could be skipped, if its value has been already exported as mentioned below:

```shell
export AWS_REGION='ap-south-1'(in this case)
export AWS_ACCESS_KEY=XXXX
export AWS_ACCESS_SECRET_KEY=XXXX
```

3. **Security Group Ports**

```
8090/tcp   -  Platform Agent               Master & Worker
10250/tcp  -  kubelet API server           Master & Worker
10255/tcp  -  kubelet API server           Master & Worker
              (read only access)
6443/tcp   -  Kubernetes API server        Master
8472/udp   -  Flannel overlay network      Master & Worker
```

— — — — —

#### <ins>vars : main.yml</ins>

```yaml
---
# vars file for aws_ec2_setup
aws_access_key: "XXXX"
aws_secret_key: "XXXX"
region: "ap-south-1"
vpc_name: "ansible_vpc"
cidr_block_vpc: "10.0.0.0/16"
igw_name: "ansible_igw"
cidr_subnet: "10.0.1.0/24"
subnet_name: "ansible_subnet"
route_table_name: "ansible_route_table"
sg_name_master: "master_sg"
sg_name_worker: "worker_sg"
instance_type: "t2.micro"
key_name: "bastion"
instance_count: 2
image_id: "ami-068d43a544160b7ef"
availability_zone: "ap-south-1a"
```

— — — — — — — — — — — — — — — — — — — — — — — — —

### Part 3 : Ansible Role (k8s_master & k8s_worker)

#### <ins>tasks : main.yml</ins>
<br>

```yaml
---
# tasks file for k8s_master
- name: Docker Installation
  package:
    name: docker
    state: present
    
- name: Starting Docker Service
  service:
    name: docker
    state: started
    enabled: yes
    
- name: Kubernetes Repo Creation
  yum_repository:
    baseurl: https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
    name: kubernetes
    description: kubernetes
    enabled: true
    gpgcheck: yes
    repo_gpgcheck: yes
    gpgkey:
      - https://packages.cloud.google.com/yum/doc/yum-key.gpg
      - https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    exclude:
      - kubelet
      - kubeadm
      - kubectl
  register: indicator
  
- name: Installation of kubeadm, kubelet and kubectl
  yum:
    name:
      - kubelet
      - kubeadm
      - kubectl
    disable_excludes: kubernetes
    state: present
    
- name: Starting kubelet Service
  service:
    name: kubelet
    state: started
    enabled: yes
    
- name: Pulling required images using kubeadm
  command: "kubeadm config images pull"
  
- name: Configuring Docker Cgroup
  template:
    src: k8s_master/templates/daemon.json
    dest: /etc/docker/daemon.json
    
- name: Restarting Docker Service
  service:
    name: docker
    state: restarted
    enabled: yes
    
- name: Installation of iproute-tc
  yum:
    name: iproute-tc
    state: present
    
- name: Copying k8s.conf to /etc/sysctl.d
  copy:
    src: k8s_master/files/k8s.conf
    dest: /etc/sysctl.d/k8s.conf
  register: sysctl_status
      
- name: Reading values from all the system directories
  command: "sysctl --system"
  when: sysctl_status.changed != false
  
- name: Initializing k8s control plane
  shell: 'kubeadm init --pod-network-cidr="{{ pod_network_cidr }}" --ignore-preflight-errors=NumCPU --ignore-preflight-errors=Mem'
  when: indicator.changed != false
  
- name: Creating .kube directory
  file:
    path: $HOME/.kube
    state: directory
  register: directory_status
  
- name: Copying admin credentials to config file in .kube directory
  copy:
    src: /etc/kubernetes/admin.conf
    dest: $HOME/.kube/config
    remote_src: yes
    
- name: Obtaining User ID
  command: "id -un"
  register: user
  
- name: Obtaining Group ID
  command: "id -ng"
  register: group
  
- name: Changing the file ownership of config file in .kube directory
  file:
    path: $HOME/.kube/config
    owner: "{{ user.stdout }}"
    group: "{{ group.stdout }}"
    
- name: Clearing RAM Memory Cache, Buffer and Swap Space
  command: "echo 3 > /proc/sys/vm/drop_caches"
  
- name: Flannel Setup
  command: "kubectl apply  -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml"
  
- name: Generating token for establishing connectivity between master and worker node
  command: "kubeadm token create  --print-join-command"
  register: token
  
- name: Generation of dummy host to pass the token to the worker nodes
  add_host:
    name: "Token_Pass_Host"
    token_pass: "{{ token }}"
```

<p align="center"><b>k8s_master</b></p><br>

#### _Steps_ : **k8s_master**

- _Install Docker and start the corresponding service._
- _Generate yum repository for Kubernetes and then install components like kubeadm, kubelet and kubectl. Enable kubelet service._
- _Use **kubeadm config image pull** to install images required for Kubernetes cluster._
- _Change the cgroup in **/etc/docker/daemon.json** file to systemd as kubelet manages kubeadm as a systemd service. Restart Docker service to apply the changes._
- _**iproute-tc** should be installed for traffic control in Kubernetes cluster._
- _In the file i.e., **/etc/sysctl.d/k8s.conf**, the parameters within it needs to set to 1 to enable bridged traffic. Use the command **sysctl --system** to apply the changes._
- _Initialize the **control plane** using kubeadm(it generates token that could be used by worker node to connect to master node)._
- _Create **.kube** directory and set up configuration file within it. Then, change the ownership of configuration file to the user and the group._
- _Clear RAM memory cache, Buffer and Swap spaces using **echo 3 > /proc/sys/vm/drop_caches** due to higher memory usage in Kubernetes cluster._
- _Overlay network setup is done on the Kubernetes cluster using **Flannel**, and it could be installed from the corresponding GitHub repository._
- _Token could be generated explicitly using **kubeadm token create --print-join-command** command and it should be passed to worker node’s Ansible role using the concept of Dummy Host._

<br><br>

```yaml
---
# tasks file for k8s_worker
- name: Obtaining the Token received from Master Node
  shell: echo "{{ hostvars['Token_Pass_Host']['token_pass']['stdout']}}"
  register: token_received
  
- name: Docker Installation
  package:
    name: docker
    state: present
    
- name: Starting Docker Service
  service:
    name: docker
    state: started
    enabled: yes
    
- name: Kubernetes Repo Creation
  yum_repository:
    baseurl: https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
    name: kubernetes
    description: kubernetes
    enabled: true
    gpgcheck: yes
    repo_gpgcheck: yes
    gpgkey:
      - https://packages.cloud.google.com/yum/doc/yum-key.gpg
      - https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    exclude:
      - kubelet
      - kubeadm
      - kubectl
  register: indicator
  
- name: Installation of kubeadm, kubelet and kubectl
  yum:
    name:
      - kubelet
      - kubeadm
      - kubectl
    disable_excludes: kubernetes
    state: present
    
- name: Starting kubelet Service
  service:
    name: kubelet
    state: started
    enabled: yes
    
- name: Configuring Docker Cgroup
  template:
    src: k8s_worker/templates/daemon.json
    dest: /etc/docker/daemon.json
    
- name: Restarting Docker Service
  service:
    name: docker
    state: restarted
    enabled: yes
    
- name: Installation of iproute-tc
  yum:
    name: iproute-tc
    state: present
    
- name: Copying k8s.conf to /etc/sysctl.d
  copy:
    src: k8s_worker/files/k8s.conf
    dest: /etc/sysctl.d/k8s.conf
  register: sysctl_status
  
- name: Reading values from all the system directories
  command: "sysctl --system"
  when: sysctl_status.changed != false
  
- name: Clearing RAM Memory Cache, Buffer and Swap Space
  command: "echo 3 > /proc/sys/vm/drop_caches"
  
- name: Executing token command to establish connectivity with Master Node
  command: "{{ token_received[\"stdout\"] }}"
  when: indicator.changed != false
```

<p align="center"><b>k8s_worker</b></p><br>

#### _Steps_ : **k8s_worker**

- _Obtain the token from the Dummy Host i.e.,**Token_Pass_Host** created in previous role i.e., k8s_master:tasks/main.yml._
- _Install Docker and start the corresponding service._
- _Generate yum repository for Kubernetes and then install components like kubeadm, kubelet and kubectl. Enable kubelet service._
- _Change the cgroup in **/etc/docker/daemon.json** file to systemd as kubelet manages kubeadm as a systemd service. Restart Docker service to apply the changes._
- _**iproute-tc** should be installed for traffic control in Kubernetes cluster._
- _In the file i.e., **/etc/sysctl.d/k8s.conf**, the parameters within it needs to set to 1 to enable bridged traffic. Use the command **sysctl --system** to apply the changes._
- _Clear RAM memory cache, Buffer and Swap spaces using **echo 3 > /proc/sys/vm/drop_caches** due to higher memory usage in Kubernetes cluster._
- _Execute the token obtained from previous role i.e., k8s_master:tasks/main.yml._

— — — — —

#### <ins>vars : main.yml</ins>
<br>

```yaml
---
# vars file for k8s_master
cgroup: "systemd"
pod_network_cidr: "10.240.0.0/16"
```

<p align="center"><b>k8s_master</b></p><br>

<br>

```yaml
---
# vars file for k8s_worker
cgroup: "systemd"
```

<p align="center"><b>k8s_worker</b></p><br>

— — — — —

#### <ins>files (k8s_master & k8s_worker)</ins>
<br>

```
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```

<p align="center"><b>k8s.conf</b></p><br>

— — — — —

#### <ins>templates (k8s_master & k8s_worker)</ins>
<br>

```json
{
  "exec-opts": ["native.cgroupdriver={{ cgroup }}"]
}
```

<p align="center"><b>daemon.json</b></p><br>

<br>
<p align="center"><b>. . .</b></p><br>

_**Note**_:<br>
_The above Ansible roles could be executed in **RedHat Linux** only._

<h2>Thank You :smiley:<h2>

[![Linkedin](https://i.stack.imgur.com/gVE0j.png) LinkedIn](https://www.linkedin.com/in/satyam-singh-95a266182)
