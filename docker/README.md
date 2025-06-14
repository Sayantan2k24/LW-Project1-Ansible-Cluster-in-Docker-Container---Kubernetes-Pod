# Setting up Ansible Cluster using Docker and RHEL 9

# Overview

This guide demonstrates how to create an Ansible control node (master) and managed nodes (slaves) using Docker containers based on Red Hat Enterprise Linux 9 (RHEL 9). The setup includes minimal required software and proper SSH configuration for Ansible management.

## Prerequisites

- Docker installed on your host system
- Basic understanding of Docker and Ansible concepts
- Access to RHEL 9 base image (redhat/ubi9)

## 1. Creating Docker Images

### Ansible Master Node Configuration

<aside>
The master node requires Python, Ansible, and SSH capabilities. Key components:

</aside>

- Base image: redhat/ubi9:latest
- Installed packages: python3, pip, ansible, openssh-server/client
- Created ansible user with sudo privileges
- Configured SSH for public key authentication

```bash
FROM redhat/ubi9:latest

# Install Python, pip, Ansible, and OpenSSH server
RUN dnf -y update && \
    dnf -y install \
        python3 \
        net-tools \
        iputils \
        python3-pip \
        openssh-clients \
        openssh-server \
        sudo \
    && dnf clean all

# Install latest Ansible via pip
RUN pip3 install --no-cache-dir ansible

# Create ansible user with sudo access
RUN useradd -d /home/ansible -s /bin/bash ansible && \
    echo "ansible ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers && \
    echo "ansible:ansible123" | chpasswd

# Create SSH directory structure
RUN mkdir -p /home/ansible/.ssh && \
    chown ansible:ansible /home/ansible/.ssh && \
    chmod 700 /home/ansible/.ssh && \
    ssh-keygen -A

# Configure SSHD for public key auth only
RUN echo "PubkeyAuthentication yes" >> /etc/ssh/sshd_config && \
    echo "PasswordAuthentication no" >> /etc/ssh/sshd_config && \
    echo "PermitRootLogin no" >> /etc/ssh/sshd_config && \
    echo "AuthorizedKeysFile .ssh/authorized_keys" >> /etc/ssh/sshd_config

# Expose SSH port
EXPOSE 22

# logged in user will be ansible
# landing directory is /home/ansible
USER ansible
WORKDIR /home/ansible

# Default command: run SSH daemon
CMD ["sudo","/usr/sbin/sshd", "-D"]
```

### Ansible Slave Node Configuration

<aside>
Slave nodes require Python and SSH capabilities for Ansible management. Key components:

</aside>

- Base image: redhat/ubi9:latest
- Installed packages: python3, openssh-server/client
- Created ansible user with sudo privileges
- Configured SSH for public key authentication

```bash
FROM redhat/ubi9:latest

# Install Python and OpenSSH server
RUN dnf -y update && \
    dnf -y install \
        python3 \
        net-tools \
        iputils \
        python3-pip \
        openssh-clients \
        openssh-server \
        sudo \
    && dnf clean all

# Create ansible user with sudo rights
RUN useradd ansible && \
    echo "ansible ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers && \
    echo "ansible:ansible123" | chpasswd

# Set up .ssh directory
RUN mkdir -p /home/ansible/.ssh && \
    chown ansible:ansible /home/ansible/.ssh && \
    chmod 700 /home/ansible/.ssh && \
    ssh-keygen -A

# Configure SSH for public key authentication only
RUN echo "PubkeyAuthentication yes" >> /etc/ssh/sshd_config && \
    echo "PasswordAuthentication no" >> /etc/ssh/sshd_config && \
    echo "PermitRootLogin no" >> /etc/ssh/sshd_config && \
    echo "AuthorizedKeysFile .ssh/authorized_keys" >> /etc/ssh/sshd_config

EXPOSE 22

# logged in user will be ansible
# landing directory is /home/ansible
USER ansible
WORKDIR /home/ansible

# Start SSH daemon
# want ansible user to run the sshd service
CMD ["sudo", "/usr/sbin/sshd", "-D"]
```

## 2. Building the Images

```bash
# Build master node image
docker build -t ansible-master-minimal:rhel9-v1 -f Dockerfile-master-rhel9 .

# Build slave node image
docker build -t ansible-slave-minimal:rhel9-v1 -f Dockerfile-slave-rhel9 .

```

## 3. Running the Containers

```bash
# Start master node
docker run -d --name ansible-master ansible-master-minimal:rhel9-v1

# Start slave nodes
docker run -d --name ansible-slave01 ansible-slave-minimal:rhel9-v1
docker run -d --name ansible-slave02 ansible-slave-minimal:rhel9-v1

```

## 4. Setting up SSH Keys

On the master node:

```bash
# Generate SSH key pair
ssh-keygen -t rsa

# View the public key
cat ~/.ssh/id_rsa.pub

```

### Copying SSH Keys to Slaves

Method 1: Using Docker CP command from host:

```bash
# Copy public key from master to host
docker cp ansible-master:/home/ansible/.ssh/id_rsa.pub .

# Copy to slave nodes
docker cp id_rsa.pub ansible-slave01:/home/ansible/.ssh/authorized_keys
docker cp id_rsa.pub ansible-slave02:/home/ansible/.ssh/authorized_keys

```

## 5. Ansible Configuration

Create ansible.cfg in the master node:

```
[defaults]
host_key_checking = False
inventory = ./inventory
remote_user = ansible
private_key_file=/home/ansible/.ssh/id_rsa
ask_pass=false

[privilege_escalation]
become=true
become_method=sudo
become_user=root
become_ask_pass=false

```

## 6. Setting up Inventory

Get IP addresses of slave nodes:

```bash
docker inspect --format '{{ .NetworkSettings.IPAddress }}' ansible-slave01
docker inspect --format '{{ .NetworkSettings.IPAddress }}' ansible-slave02

```

Create inventory file on master node:

```
[slaves]
slave01 ansible_host=172.17.0.2
slave02 ansible_host=172.17.0.4

```

## 7. Verifying the Setup

```bash
# List all hosts
ansible all --list-hosts

# Test connectivity
ansible all -m ping

# Test basic commands
ansible all -m command -a 'date'
ansible all -m command -a 'whoami'

```

## Important Notes

<aside>

- All containers use the ansible user with sudo privileges
- SSH is configured for public key authentication only
- Password authentication is disabled for security
- Root login is disabled
</aside>

This setup provides a minimal, secure environment for testing and learning Ansible automation in containers.
