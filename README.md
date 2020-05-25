# EWF Token Bridge

Installation tutorial step by step

## Table of Content
<!-- TOC -->

- [EWF Token Bridge](#ewf-token-bridge)
  - [Table of Content](#table-of-content)
  - [Required software on local machine](#required-software-on-local-machine)
  - [AWS Instance preparation](#aws-instance-preparation)
    - [AWS EC2](#aws-ec2)
    - [Volumes preparation](#volumes-preparation)
  - [AWS Instance required configuration](#aws-instance-required-configuration)
    - [Prepare directories for blockchain data](#prepare-directories-for-blockchain-data)
  - [Ansible common stack (cs)](#ansible-common-stack-cs)
    - [Ansible configuration (cs)](#ansible-configuration-cs)
      - [Variables file (cs)](#variables-file-cs)
      - [Hosts file (cs)](#hosts-file-cs)
    - [Ansible run (cs)](#ansible-run-cs)
    - [RPC logs - check in](#rpc-logs---check-in)

<!-- /TOC -->
## Required software on local machine

- git
- ansible

## AWS Instance preparation

### AWS EC2

We need pretty big instance with 3 additional ebs drives

**Type**

m5.xlarge

**Volumes**

```txt
/bridge-data - 100 GB
/parity-main - 500 GB
/parity-ewc - 300 GB
/ - 60 GB
```

**Security group**

SSH Should be restricted to specified IP's
At least we need only to expose HTTP & HTTPS ports becase whole traffic will be forwarded through nginx. In some cases security groups should be modified to fits your needs

```txt
80	tcp	0.0.0.0/0	✔
22	tcp	0.0.0.0/0	✔
443	tcp	0.0.0.0/0	✔
```

**Other requirements**

Static Public IP Address

Currently we use ubuntu 18.04 for bridge purposes
AMI ID: `ubuntu/images/hvm-ssd/ubuntu-bionic-18.04-amd64-server-20200112 (ami-03ba3948f6c37a4b0)`

### Volumes preparation

Whole process has been described in [AWS documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-using-volumes.html)

```bash
sudo mkdir /bridge-data
sudo mkdir /parity-main
sudo mkdir /parity-ewc
sudo yum install xfsprogs

sudo mkfs -t xfs /dev/nvme0n1
sudo mount /dev/nvme0n1 /parity-ewc

sudo mkfs -t xfs /dev/nvme1n1
sudo mount /dev/nvme1n1 /bridge-data

sudo mkfs -t xfs /dev/nvme2n1
sudo mount /dev/nvme2n1 /parity-main

sudo cp /etc/fstab /etc/fstab.orig
sudo lsblk -o +UUID
sudo vim /etc/fstab

# Added these lines without '#' and replace UUID's with ones outputed by above commands
# UUID=ea1086b2-f097-4edb-8ebf-1768f0d3250b  /parity-ewc  xfs  defaults,nofail  0  2
# UUID=5242378c-c216-4fb3-82b5-d725becc95cf  /bridge-data  xfs  defaults,nofail  0  2
# UUID=726d4247-605f-4220-9667-3d846a3a0fa1  /parity-main  xfs  defaults,nofail  0  2

sudo mount -a
```

Expected result example

```bash
nvme0n1     259:0    0  300G  0 disk /parity-ewc                 ea1086b2-f097-4edb-8ebf-1768f0d3250b
nvme1n1     259:1    0  100G  0 disk /bridge-data                5242378c-c216-4fb3-82b5-d725becc95cf
nvme2n1     259:2    0  500G  0 disk /parity-main                726d4247-605f-4220-9667-3d846a3a0fa1
nvme3n1     259:3    0   60G  0 disk
└─nvme3n1p1 259:4    0   60G  0 part /                           c03b791b-60ef-4ae1-82b5-5c9ab6b4d08f
```

## AWS Instance required configuration 

### Prepare directories for blockchain data

```bash
sudo chown ubuntu:ubuntu /bridge-data
sudo chown ubuntu:ubuntu /parity-main
sudo chown ubuntu:ubuntu /parity-ewc

```

## Ansible common stack (cs)

In this part we are going to install basic dependencies & common resources stack on our bridge instance.
This stack contains required RPC which should be fully synced before we can continue bridge installation.
All files related to this part are located in the `ansible-common-stack` and `resources-docker-stack` directories

### Ansible configuration (cs)

#### Variables file (cs)

Configuration file location: `/ansible-common-stack/group_vars/ewc_bridge.yml`
In this file more advanced users can adjust bridge and rpc data location if needed or overwrite every value used inside stack

#### Hosts file (cs)

To work with ansible we have to prepare inventory file where we will keep all hosts where we would like to run our playbooks
You can simply copy example file: `cp ansible-common-stack/hosts.yml.example ansible-common-stack/hosts-ewc.yml` and IP with new one associated with your AWS instance. This is necessary for ansible to make a ssh connection and run all commands.

### Ansible run (cs)

Now when everything has been prepared we can open terminal and run ansible playbooks directly from `ansible-common-stack` directory.
Additionally we provide `--private-key` flag with value to the location where we keep key for ssh connection with that instance. Remember to replace it with your own key path.

Simply run below command and watch output:
```bash
ansible-playbook -i hosts-ewc.yml site.yml --private-key=~/.ssh/id_rsa
```

**Now we wait till RPC will be fully synced**

### RPC logs - check in

For convenience we have prepared possibility to verify RPC status with ansible command as well

Edit variables file with your favourite editor: `ansible-common-stack/group_vars/ewc_bridge.yml`

Uncomment this line `check_rpc_status_only: true` and save file.

Now we can again run ansible command but instead of installation proccess we should see parity logs with sync status
```bash
ansible-playbook -i hosts-ewc.yml site.yml --private-key=~/.ssh/id_rsa
```
