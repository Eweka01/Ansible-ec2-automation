# Ansible EC2 Automation: Production-Ready Infrastructure Management

## Table of Contents
1. [Project Overview](#project-overview)
2. [Prerequisites](#prerequisites)
3. [Project Structure](#project-structure)
4. [Core Automation Tasks](#core-automation-tasks)
5. [Advanced Error Handling](#advanced-error-handling)
6. [Security Implementation](#security-implementation)
7. [Best Practices & Production Tips](#best-practices--production-tips)
8. [Troubleshooting Guide](#troubleshooting-guide)
9. [Real-World Applications](#real-world-applications)
10. [Conclusion](#conclusion)

---

## Project Overview

This project demonstrates enterprise-grade Ansible automation for AWS EC2 infrastructure management. Originally designed as an interview assessment, it showcases critical DevOps concepts including infrastructure provisioning, configuration management, and operational automation.

### 🎯 **Key Learning Objectives**

- **Infrastructure as Code (IaC)** - Automated EC2 instance provisioning
- **Configuration Management** - Secure passwordless authentication setup
- **Operational Automation** - Conditional shutdown procedures
- **Error Handling** - Robust automation with failure recovery
- **Security Best Practices** - Ansible Vault implementation
- **Production Readiness** - Real-world deployment patterns

### 🏗️ **Architecture Overview**

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Ansible       │    │      AWS         │    │   EC2 Instances │
│  Control Node   │────│     APIs         │────│                 │
│                 │    │                  │    │ • Amazon Linux  │
│ • Playbooks     │    │ • EC2 Service    │    │ • Ubuntu 1      │
│ • Inventory     │    │ • IAM Policies   │    │ • Ubuntu 2      │
│ • Vault Secrets │    │ • Key Pairs      │    │                 │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```

---

## Prerequisites

### 🔧 **Software Requirements**

```bash
# Python Virtual Environment Setup
python3 -m venv venv
source venv/bin/activate  # Linux/Mac
# venv\Scripts\activate   # Windows

# Install Core Dependencies
pip install ansible boto3

# Install AWS Collection
ansible-galaxy collection install amazon.aws
```

### ☁️ **AWS Configuration**

#### 1. IAM User Setup
```bash
# Required IAM Permissions
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:*",
                "iam:PassRole"
            ],
            "Resource": "*"
        }
    ]
}
```

#### 2. Key Pair Creation
```bash
# Create EC2 Key Pair
aws ec2 create-key-pair --key-name key-p3 --query 'KeyMaterial' --output text > key-p3.pem
chmod 400 key-p3.pem
```

#### 3. Security Group Configuration
```bash
# Default security group with SSH access
# Port 22 (SSH) - Source: Your IP
# Port 80 (HTTP) - Source: 0.0.0.0/0
# Port 443 (HTTPS) - Source: 0.0.0.0/0
```

---

## Project Structure

```
Ansible-ec2-automation/
├── 📁 Ec2 Playbook/                    # Core automation playbooks
│   ├── 📄 ec2_create.yaml             # EC2 provisioning with loops
│   ├── 📄 ec2_shutdown.yaml           # Conditional shutdown automation
│   ├── 📄 inventory.ini               # Target hosts configuration
│   ├── 📄 README.md                   # Task descriptions
│   ├── 📄 vault.pass                  # Vault password file
│   └── 📁 group_vars/
│       └── 📁 all/
│           └── 📄 pass.yml            # Encrypted AWS credentials
├── 📁 error-handling/                  # Advanced error handling examples
│   ├── 📄 main.yaml                   # Comprehensive error handling
│   └── 📄 inventory.ini               # Test environment inventory
├── 📁 venv/                           # Python virtual environment
└── 📄 README.md                       # Project overview
```

---

## Core Automation Tasks

### 🚀 **Task 1: EC2 Instance Provisioning with Loops**

#### **Challenge Overview**
Create three EC2 instances with different configurations:
- 2 instances with Ubuntu distribution
- 1 instance with Amazon Linux distribution

#### **Key Concepts Demonstrated**

##### 1. **Connection Type: Local**
```yaml
- hosts: localhost
  connection: local
```
**Why local connection?** AWS is a cloud platform, not a traditional server. Ansible must execute AWS API calls directly from the control node.

##### 2. **Ansible Idempotency**
```yaml
# ❌ WRONG - Will only create one instance
name: "ansible-instance"  # Same name for all instances

# ✅ CORRECT - Creates three unique instances
name: "{{ item.name }}"   # Unique name per iteration
```

**Critical Learning**: Ansible's idempotency prevents duplicate resources. Using identical names results in only one instance being created.

#### **Complete Provisioning Playbook**

```yaml
---
- hosts: localhost
  connection: local

  tasks:
  - name: Create EC2 instances with different configurations
    amazon.aws.ec2_instance:
      name: "{{ item.name }}"
      key_name: "key-p3"
      instance_type: t2.micro
      security_group: default
      region: us-east-1
      aws_access_key: "{{ec2_access_key}}"
      aws_secret_key: "{{ec2_secret_key}}"
      network:
        assign_public_ip: true
      image_id: "{{ item.image }}"
      tags:
        environment: "{{ item.name }}"
        project: "ansible-automation"
        managed_by: "ansible"
    loop:
      - { image: "ami-052064a798f08f0d3", name: "manage-node-1" } # Amazon Linux 2
      - { image: "ami-0360c520857e3138f", name: "manage-node-2" } # Ubuntu 20.04
      - { image: "ami-0360c520857e3138f", name: "manage-node-3" } # Ubuntu 20.04
```

#### **AMI ID Discovery**

```bash
# Method 1: AWS Console
# EC2 Dashboard → Launch Instance → Select AMI → Copy AMI ID

# Method 2: AWS CLI
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*" \
  --query 'Images[*].[ImageId,Name,CreationDate]' \
  --output table

# Method 3: Ansible Facts
- name: Get latest Amazon Linux 2 AMI
  amazon.aws.ec2_ami_info:
    owners: amazon
    filters:
      name: "amzn2-ami-hvm-*"
      architecture: x86_64
      virtualization-type: hvm
    region: us-east-1
  register: amazon_linux_amis
```

#### **Execution & Verification**

```bash
# Run the provisioning playbook
ansible-playbook "Ec2 Playbook/ec2_create.yaml" --vault-password-file "Ec2 Playbook/vault.pass"

# Verify instances in AWS Console
# Expected: 3 instances with unique names and different AMIs
```

**Expected Output:**
```
PLAY [localhost] ***************************************************************

TASK [Gathering Facts] *********************************************************
ok: [localhost]

TASK [Create EC2 instances with different configurations] **********************
changed: [localhost] => (item={'image': 'ami-052064a798f08f0d3', 'name': 'manage-node-1'})
changed: [localhost] => (item={'image': 'ami-0360c520857e3138f', 'name': 'manage-node-2'})
changed: [localhost] => (item={'image': 'ami-0360c520857e3138f', 'name': 'manage-node-3'})

PLAY RECAP *********************************************************************
localhost                  : ok=2    changed=3    unreachable=0    failed=0
```

---

### 🔐 **Task 2: Passwordless Authentication Setup**

#### **Security Architecture**

```
┌─────────────────┐    SSH Key Exchange    ┌─────────────────┐
│   Control Node  │◄─────────────────────►│   EC2 Instances │
│                 │                        │                 │
│ • Private Key   │                        │ • Public Key    │
│ • Ansible       │                        │ • SSH Service   │
│ • Playbooks     │                        │ • User Accounts │
└─────────────────┘                        └─────────────────┘
```

#### **SSH Key-Based Authentication**

##### 1. **Prepare Key Pair**
```bash
# Set correct permissions
chmod 400 key-p3.pem

# Verify key format
file key-p3.pem
# Expected: key-p3.pem: PEM RSA private key
```

##### 2. **Automated Key Distribution**
```bash
# For Amazon Linux instances
ssh-copy-id -f -i key-p3.pem ec2-user@<AMAZON_LINUX_IP>

# For Ubuntu instances
ssh-copy-id -f -i key-p3.pem ubuntu@<UBUNTU_IP>

# Verify connection
ssh ec2-user@<AMAZON_LINUX_IP> "echo 'Connection successful'"
ssh ubuntu@<UBUNTU_IP> "echo 'Connection successful'"
```

##### 3. **Inventory Configuration**
```ini
[all]
ec2-user@52.201.113.106    # Amazon Linux instance
ubuntu@35.168.17.47        # Ubuntu instance 1
ubuntu@54.158.236.221      # Ubuntu instance 2

[amazon_linux]
ec2-user@52.201.113.106

[ubuntu_servers]
ubuntu@35.168.17.47
ubuntu@54.158.236.221
```

#### **Alternative: Ansible-Based Key Setup**

```yaml
---
- hosts: all
  become: true
  tasks:
    - name: Ensure SSH directory exists
      ansible.builtin.file:
        path: "{{ ansible_env.HOME }}/.ssh"
        state: directory
        mode: '0700'
        owner: "{{ ansible_user_id }}"

    - name: Add SSH public key to authorized_keys
      ansible.posix.authorized_key:
        user: "{{ ansible_user_id }}"
        key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
        state: present
        mode: '0600'
```

---

### ⚡ **Task 3: Conditional Shutdown Automation**

#### **Business Scenario**
Implement automated shutdown for Ubuntu instances only, preserving Amazon Linux instances for critical services.

#### **Understanding Ansible Facts**

```yaml
# Debug: View all gathered facts
- name: Display system information
  ansible.builtin.debug:
    var: ansible_facts
```

**Key OS-related facts:**
```yaml
ansible_facts['os_family']        # RedHat, Debian, etc.
ansible_facts['distribution']     # Ubuntu, Amazon, CentOS
ansible_facts['distribution_version']  # 20.04, 2, 8
ansible_facts['architecture']     # x86_64, arm64
```

#### **Conditional Shutdown Implementation**

```yaml
---
- hosts: all
  become: true

  tasks:
    - name: Shutdown Ubuntu instances only
      ansible.builtin.command: /sbin/shutdown -t now
      when: ansible_facts['os_family'] == "Debian"
      
    - name: Log shutdown action
      ansible.builtin.debug:
        msg: "Shutting down {{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}"
      when: ansible_facts['os_family'] == "Debian"
```

#### **Advanced Conditional Logic**

```yaml
---
- hosts: all
  become: true

  tasks:
    - name: Graceful shutdown for Ubuntu instances
      ansible.builtin.command: /sbin/shutdown -h +5 "Scheduled maintenance shutdown"
      when: 
        - ansible_facts['os_family'] == "Debian"
        - ansible_facts['distribution'] == "Ubuntu"
        
    - name: Immediate shutdown for development instances
      ansible.builtin.command: /sbin/shutdown -t now
      when:
        - ansible_facts['os_family'] == "Debian"
        - inventory_hostname in groups['development']
        
    - name: Skip shutdown for production instances
      ansible.builtin.debug:
        msg: "Skipping shutdown for production instance: {{ inventory_hostname }}"
      when: inventory_hostname in groups['production']
```

#### **Execution & Monitoring**

```bash
# Run conditional shutdown
ansible-playbook -i "Ec2 Playbook/inventory.ini" "Ec2 Playbook/ec2_shutdown.yaml" --vault-password-file "Ec2 Playbook/vault.pass"

# Monitor instance states
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,State.Name,Tags[?Key==`Name`].Value|[0]]' --output table
```

**Expected Output:**
```
PLAY [all] *********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [ec2-user@52.201.113.106]    # Amazon Linux - facts gathered
ok: [ubuntu@35.168.17.47]        # Ubuntu - facts gathered
ok: [ubuntu@54.158.236.221]      # Ubuntu - facts gathered

TASK [Shutdown Ubuntu instances only] ******************************************
skipping: [ec2-user@52.201.113.106]  # Amazon Linux - condition not met
changed: [ubuntu@35.168.17.47]       # Ubuntu - shutdown initiated
changed: [ubuntu@54.158.236.221]     # Ubuntu - shutdown initiated

PLAY RECAP *********************************************************************
ec2-user@52.201.113.106    : ok=1    changed=0    skipped=1
ubuntu@35.168.17.47        : ok=1    changed=1
ubuntu@54.158.236.221      : ok=1    changed=1
```

---

## Advanced Error Handling

### 🛡️ **Production-Grade Error Management**

The `error-handling/main.yaml` demonstrates enterprise-level error handling patterns:

```yaml
---
- hosts: webservers
  become: true

  tasks:
    # 1. Package Updates with Error Tolerance
    - name: Update security packages
      ansible.builtin.apt:
        name: "{{ item }}"
        state: latest
      loop:
        - openssh
        - openssl
      ignore_errors: yes  # Continue execution if package update fails

    # 2. Dependency Installation
    - name: Install required system packages
      ansible.builtin.apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
          - software-properties-common
        state: present

    # 3. Conditional File Check with Custom Failure Logic
    - name: Verify system integrity
      ansible.builtin.command: ls /tmp/this_should_not_be_here
      register: result
      failed_when:
        - result.rc == 0
        - '"No such" not in result.stderr'

    # 4. Docker Installation with Repository Setup
    - name: Add Docker GPG key
      ansible.builtin.apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present

    - name: Add Docker repository
      ansible.builtin.apt_repository:
        repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable
        state: present

    - name: Install Docker CE
      ansible.builtin.apt:
        name: docker-ce
        state: present
        update_cache: yes

    # 5. Verification with Error Handling
    - name: Verify Docker installation
      ansible.builtin.command: docker --version
      register: output
      ignore_errors: yes
```

### 🔧 **Error Handling Patterns**

#### 1. **Graceful Degradation**
```yaml
- name: Install optional package
  ansible.builtin.apt:
    name: optional-package
    state: present
  ignore_errors: yes
```

#### 2. **Custom Failure Conditions**
```yaml
- name: Check service status
  ansible.builtin.command: systemctl status nginx
  register: service_status
  failed_when:
    - service_status.rc != 0
    - '"inactive" in service_status.stdout'
```

#### 3. **Retry Logic**
```yaml
- name: Wait for service to start
  ansible.builtin.wait_for:
    port: 80
    delay: 10
    timeout: 60
  retries: 3
  delay: 5
```

#### 4. **Conditional Recovery**
```yaml
- name: Restart service if failed
  ansible.builtin.systemd:
    name: nginx
    state: restarted
  when: service_status.failed
```

---

## Security Implementation

### 🔒 **Ansible Vault Configuration**

#### 1. **Vault Password Management**
```bash
# Create secure vault password
echo "your-super-secure-password" | base64 > "Ec2 Playbook/vault.pass"

# Verify password file
chmod 600 "Ec2 Playbook/vault.pass"
```

#### 2. **Encrypted Credentials Storage**
```bash
# Create encrypted variables file
ansible-vault create "Ec2 Playbook/group_vars/all/pass.yml" --vault-password-file "Ec2 Playbook/vault.pass"
```

**Encrypted content structure:**
```yaml
# In group_vars/all/pass.yml (encrypted)
ec2_access_key: "AKIAIOSFODNN7EXAMPLE"
ec2_secret_key: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
aws_region: "us-east-1"
```

#### 3. **Vault Usage in Playbooks**
```yaml
aws_access_key: "{{ec2_access_key}}"
aws_secret_key: "{{ec2_secret_key}}"
region: "{{aws_region}}"
```

### 🛡️ **Security Best Practices**

#### 1. **Least Privilege IAM**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:RunInstances",
                "ec2:TerminateInstances",
                "ec2:CreateTags",
                "ec2:DescribeImages"
            ],
            "Resource": "*"
        }
    ]
}
```

#### 2. **Network Security**
```yaml
- name: Configure security group
  amazon.aws.ec2_security_group:
    name: ansible-managed-sg
    description: Security group for Ansible-managed instances
    rules:
      - proto: tcp
        ports:
          - 22
        cidr_ip: "{{ ansible_default_ipv4.address }}/32"  # Only control node
      - proto: tcp
        ports:
          - 80
        cidr_ip: 0.0.0.0/0
```

#### 3. **Audit Logging**
```yaml
- name: Enable CloudTrail logging
  amazon.aws.cloudtrail:
    name: ansible-audit-trail
    s3_bucket_name: "{{ audit_logs_bucket }}"
    is_multi_region_trail: true
    include_global_service_events: true
```

---

## Best Practices & Production Tips

### 🏭 **Production Deployment Patterns**

#### 1. **Environment-Specific Configurations**
```yaml
# group_vars/production/vars.yml
instance_type: t3.medium
min_instances: 3
max_instances: 10
backup_enabled: true

# group_vars/development/vars.yml
instance_type: t2.micro
min_instances: 1
max_instances: 3
backup_enabled: false
```

#### 2. **Dynamic Inventory**
```yaml
# aws_ec2.yml
plugin: aws_ec2
regions:
  - us-east-1
  - us-west-2
filters:
  tag:Environment: "{{ target_environment }}"
keyed_groups:
  - key: tags.Environment
    prefix: env
  - key: instance_type
    prefix: type
  - key: tags.Role
    prefix: role
```

#### 3. **Rolling Updates**
```yaml
- name: Rolling update of application servers
  amazon.aws.ec2_instance:
    name: "app-server-{{ item }}"
    state: running
    image_id: "{{ new_ami_id }}"
  serial: 1  # Update one instance at a time
  loop: "{{ range(1, app_server_count + 1) | list }}"
```

#### 4. **Health Checks**
```yaml
- name: Wait for instance to be ready
  amazon.aws.ec2_instance_info:
    instance_ids: "{{ new_instance.instance_ids }}"
  register: instance_info
  until: instance_info.instances[0].state.name == "running"
  retries: 30
  delay: 10

- name: Verify application health
  ansible.builtin.uri:
    url: "http://{{ item.public_ip_address }}/health"
    method: GET
  loop: "{{ instance_info.instances }}"
  register: health_check
  until: health_check.status == 200
  retries: 10
  delay: 5
```

### 📊 **Monitoring & Observability**

#### 1. **CloudWatch Integration**
```yaml
- name: Install CloudWatch agent
  ansible.builtin.apt:
    name: amazon-cloudwatch-agent
    state: present

- name: Configure CloudWatch agent
  ansible.builtin.copy:
    content: |
      {
        "metrics": {
          "namespace": "AnsibleManaged/EC2",
          "metrics_collected": {
            "cpu": {
              "measurement": ["cpu_usage_idle", "cpu_usage_iowait"],
              "metrics_collection_interval": 60
            }
          }
        }
      }
    dest: /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

#### 2. **Log Aggregation**
```yaml
- name: Configure log forwarding
  ansible.builtin.lineinfile:
    path: /etc/rsyslog.conf
    line: "*.* @@{{ log_server }}:514"
    state: present
  notify: restart rsyslog

- name: Install log rotation
  ansible.builtin.cron:
    name: "Rotate Ansible logs"
    job: "logrotate /etc/logrotate.d/ansible"
    minute: "0"
    hour: "2"
```

---

## Troubleshooting Guide

### 🔍 **Common Issues & Solutions**

#### 1. **Authentication Failures**
```bash
# Issue: Permission denied (publickey)
# Solution: Verify SSH key setup
ssh -i key-p3.pem -v ec2-user@<IP>

# Debug SSH connection
ssh -vvv -i key-p3.pem ec2-user@<IP>
```

#### 2. **AMI ID Errors**
```bash
# Issue: Invalid AMI ID
# Solution: Verify AMI exists in your region
aws ec2 describe-images --image-ids ami-052064a798f08f0d3 --region us-east-1

# Find correct AMI for your region
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*" \
  --region us-west-2
```

#### 3. **Vault Decryption Issues**
```bash
# Issue: Vault decryption failed
# Solution: Verify vault password file
ansible-vault view "Ec2 Playbook/group_vars/all/pass.yml" --vault-password-file "Ec2 Playbook/vault.pass"

# Test vault password
echo "your-password" | base64 -d
```

#### 4. **Network Connectivity**
```bash
# Issue: Cannot reach instances
# Solution: Check security groups
aws ec2 describe-security-groups --group-names default

# Verify instance state
aws ec2 describe-instances --instance-ids i-1234567890abcdef0
```

### 🛠️ **Debugging Techniques**

#### 1. **Verbose Execution**
```bash
# Maximum verbosity
ansible-playbook playbook.yaml -vvv

# Check mode (dry run)
ansible-playbook playbook.yaml --check

# Step-by-step execution
ansible-playbook playbook.yaml --step
```

#### 2. **Fact Gathering**
```yaml
- name: Debug specific facts
  ansible.builtin.debug:
    var: ansible_facts['os_family']

- name: Debug all facts
  ansible.builtin.debug:
    var: ansible_facts
```

#### 3. **Connection Testing**
```bash
# Test inventory connectivity
ansible all -i inventory.ini -m ping

# Test specific host
ansible ec2-user@52.201.113.106 -i inventory.ini -m ping
```

---

## Real-World Applications

### 🏢 **Enterprise Use Cases**

#### 1. **Multi-Environment Management**
```yaml
# environments/production.yml
- name: Deploy production infrastructure
  hosts: localhost
  connection: local
  vars:
    environment: production
    instance_count: 5
    instance_type: t3.large
  tasks:
    - include_tasks: tasks/create_instances.yml
    - include_tasks: tasks/configure_security.yml
    - include_tasks: tasks/deploy_application.yml
```

#### 2. **Auto-Scaling Groups**
```yaml
- name: Create Auto Scaling Group
  amazon.aws.autoscaling_group:
    name: "{{ app_name }}-asg"
    launch_template_name: "{{ app_name }}-lt"
    min_size: 2
    max_size: 10
    desired_capacity: 3
    vpc_zone_identifier: "{{ subnet_ids }}"
    health_check_type: ELB
    health_check_grace_period: 300
```

#### 3. **Disaster Recovery**
```yaml
- name: Backup critical instances
  amazon.aws.ec2_snapshot:
    instance_id: "{{ item }}"
    description: "Daily backup - {{ ansible_date_time.iso8601 }}"
  loop: "{{ critical_instances }}"
  register: backup_snapshots

- name: Cross-region replication
  amazon.aws.ec2_snapshot_copy:
    source_region: us-east-1
    source_snapshot_id: "{{ item.snapshot_id }}"
    region: us-west-2
  loop: "{{ backup_snapshots.results }}"
```

#### 4. **Cost Optimization**
```yaml
- name: Schedule instance shutdown
  amazon.aws.ec2_instance:
    instance_ids: "{{ item }}"
    state: stopped
  loop: "{{ development_instances }}"
  when: ansible_date_time.hour >= 18  # After 6 PM

- name: Schedule instance startup
  amazon.aws.ec2_instance:
    instance_ids: "{{ item }}"
    state: running
  loop: "{{ development_instances }}"
  when: ansible_date_time.hour == 8   # At 8 AM
```

### 🚀 **CI/CD Integration**

#### 1. **GitHub Actions Workflow**
```yaml
# .github/workflows/deploy.yml
name: Deploy Infrastructure
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Setup Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.9'
      - name: Install dependencies
        run: |
          pip install ansible boto3
          ansible-galaxy collection install amazon.aws
      - name: Deploy infrastructure
        run: |
          ansible-playbook "Ec2 Playbook/ec2_create.yaml" \
            --vault-password-file "Ec2 Playbook/vault.pass"
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

#### 2. **Jenkins Pipeline**
```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Deploy') {
            steps {
                sh '''
                    pip install ansible boto3
                    ansible-galaxy collection install amazon.aws
                    ansible-playbook "Ec2 Playbook/ec2_create.yaml" \
                        --vault-password-file "Ec2 Playbook/vault.pass"
                '''
            }
        }
    }
}
```

---

## Conclusion

### 🎯 **Key Achievements**

This project successfully demonstrates:

1. **Infrastructure as Code** - Automated EC2 provisioning with loops
2. **Configuration Management** - Secure passwordless authentication
3. **Operational Automation** - Conditional shutdown procedures
4. **Error Handling** - Production-grade failure recovery
5. **Security Implementation** - Ansible Vault and best practices
6. **Production Readiness** - Real-world deployment patterns

### 🚀 **Next Steps for Production**

1. **Implement Monitoring** - CloudWatch, Prometheus, Grafana
2. **Add Backup Automation** - Automated snapshots and cross-region replication
3. **Create Rollback Procedures** - Blue-green deployments
4. **Implement CI/CD Pipelines** - Automated testing and deployment
5. **Add Compliance Checks** - Security scanning and policy enforcement

### 📚 **Learning Resources**

- [Ansible Official Documentation](https://docs.ansible.com/)
- [AWS Ansible Collection](https://docs.ansible.com/ansible/latest/collections/amazon/aws/)
- [Infrastructure as Code Best Practices](https://martinfowler.com/bliki/InfrastructureAsCode.html)
- [DevOps Culture and Practices](https://aws.amazon.com/devops/what-is-devops/)

### 🤝 **Contributing**

This project serves as a foundation for:
- **Ansible automation**
- **Understanding AWS integration**
- **Implementing DevOps practices**
- **Building production-ready infrastructure**

---

**🌟 Ready to automate your infrastructure? Start with this project and scale to enterprise-grade solutions!**

*Happy Automating! 🚀*
