# Complete Guide to Ansible EC2 Automation: From Zero to Production-Ready Infrastructure

## Table of Contents
1. [Project Overview](#project-overview)
2. [Prerequisites](#prerequisites)
3. [Project Structure](#project-structure)
4. [Task 1: EC2 Instance Provisioning with Loops](#task-1-ec2-instance-provisioning-with-loops)
5. [Task 2: Passwordless Authentication Setup](#task-2-passwordless-authentication-setup)
6. [Task 3: Conditional Shutdown Automation](#task-3-conditional-shutdown-automation)
7. [Error Handling and Best Practices](#error-handling-and-best-practices)
8. [Security Considerations](#security-considerations)
9. [Advanced Features](#advanced-features)
10. [Troubleshooting](#troubleshooting)
11. [Conclusion](#conclusion)

---

## Project Overview

This project demonstrates a real-world Ansible automation scenario that was originally presented as an interview assignment. It showcases three critical aspects of infrastructure automation:

1. **Infrastructure Provisioning** - Creating multiple EC2 instances using Ansible loops
2. **Configuration Management** - Setting up secure passwordless authentication
3. **Operational Automation** - Implementing conditional shutdown procedures

The project emphasizes key Ansible concepts including:
- **Loops** for repetitive tasks
- **Conditionals** for decision-making
- **Idempotency** for safe re-execution
- **Security** through Ansible Vault
- **Error Handling** for robust automation

---

## Prerequisites

### Software Requirements
```bash
# Install Ansible
pip install ansible

# Install AWS SDK for Python
pip install boto3

# Install Ansible AWS Collection
ansible-galaxy collection install amazon.aws
```

### AWS Setup
1. **Create IAM User** with EC2 permissions:
   - Go to AWS IAM Console
   - Create user: `ansible-user`
   - Attach policy: `AmazonEC2FullAccess`
   - Generate Access Key and Secret Key

2. **Create Key Pair**:
   - Go to EC2 Console → Key Pairs
   - Create new key pair: `key-p3`
   - Download the `.pem` file

### Project Dependencies
```bash
# Virtual environment setup (recommended)
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install ansible boto3
```

---

## Project Structure

```
Ansible-ec2-automation/
├── day-07/                          # Main project directory
│   ├── ec2_create.yaml             # EC2 provisioning playbook
│   ├── ec2_shutdown.yaml           # Conditional shutdown playbook
│   ├── inventory.ini               # Target hosts inventory
│   ├── group_vars/
│   │   └── all/
│   │       └── pass.yml            # Encrypted AWS credentials
│   ├── vault.pass                  # Vault password file
│   └── README.md                   # Project documentation
├── Day-08/                         # Error handling examples
│   ├── 01-error-handling.yaml     # Error handling playbook
│   └── inventory.ini               # Test inventory
├── error-handling/                 # Advanced error handling
│   ├── main.yaml                   # Comprehensive error handling
│   └── inventory.ini               # Production inventory
└── venv/                          # Python virtual environment
```

---

## Task 1: EC2 Instance Provisioning with Loops

### Understanding the Challenge

The first task requires creating **three EC2 instances** with different configurations:
- 2 instances with Ubuntu distribution
- 1 instance with Amazon Linux distribution

### Key Learning Points

#### 1. Connection Type: Local
```yaml
- hosts: localhost
  connection: local
```
**Why local connection?** AWS is a cloud platform, not a server you can SSH into. The Ansible control node must execute AWS API calls directly.

#### 2. Ansible Idempotency
This is a **critical concept** that many developers miss. Ansible's idempotency means:
- If the desired state already exists, Ansible won't recreate it
- This prevents duplicate resources and ensures safe re-execution

**The Interview Trap**: If you use the same instance name for all three instances, only one will be created due to idempotency!

### Complete EC2 Creation Playbook

```yaml
---
- hosts: localhost
  connection: local

  tasks:
  - name: Create EC2 instances
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
    loop:
      - { image: "ami-052064a798f08f0d3", name: "manage-node-1" } # Amazon Linux
      - { image: "ami-0360c520857e3138f", name: "manage-node-2" } # Ubuntu
      - { image: "ami-0360c520857e3138f", name: "manage-node-3" } # Ubuntu
```

### Breaking Down the Loop

```yaml
loop:
  - { image: "ami-xxx", name: "manage-node-1" }
  - { image: "ami-yyy", name: "manage-node-2" }
  - { image: "ami-yyy", name: "manage-node-3" }
```

**Variables in the loop:**
- `{{ item.image }}` - AMI ID for each instance
- `{{ item.name }}` - Unique name for each instance

### Finding AMI IDs

```bash
# Method 1: AWS Console
# Go to EC2 → Launch Instance → Select AMI → Copy AMI ID

# Method 2: AWS CLI
aws ec2 describe-images --owners amazon --filters "Name=name,Values=amzn2-ami-hvm-*" --query 'Images[*].[ImageId,Name]' --output table
```

### Execution

```bash
# Run the playbook
ansible-playbook ec2_create.yaml --vault-password-file vault.pass
```

**Expected Output:**
```
PLAY [localhost] ***************************************************************

TASK [Gathering Facts] *********************************************************
ok: [localhost]

TASK [Create EC2 instances] ****************************************************
changed: [localhost] => (item={'image': 'ami-052064a798f08f0d3', 'name': 'manage-node-1'})
changed: [localhost] => (item={'image': 'ami-0360c520857e3138f', 'name': 'manage-node-2'})
changed: [localhost] => (item={'image': 'ami-0360c520857e3138f', 'name': 'manage-node-3'})

PLAY RECAP *********************************************************************
localhost                  : ok=2    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

---

## Task 2: Passwordless Authentication Setup

### Why Passwordless Authentication?

Before we can manage our EC2 instances with Ansible, we need to establish secure, passwordless SSH connections. This eliminates the need to store passwords and enables automated management.

### SSH Key-Based Authentication

#### Step 1: Prepare Your Key Pair
```bash
# Ensure your .pem file has correct permissions
chmod 400 key-p3.pem
```

#### Step 2: Set Up Passwordless Authentication
```bash
# For Amazon Linux instances
ssh-copy-id -f -i key-p3.pem ec2-user@<AMAZON_LINUX_IP>

# For Ubuntu instances  
ssh-copy-id -f -i key-p3.pem ubuntu@<UBUNTU_IP>
```

#### Step 3: Test Connection
```bash
# Test without specifying key file
ssh ec2-user@<AMAZON_LINUX_IP>
ssh ubuntu@<UBUNTU_IP>
```

### Inventory File Configuration

```ini
[all]
ec2-user@52.201.113.106    # Amazon Linux instance
ubuntu@35.168.17.47        # Ubuntu instance 1
ubuntu@54.158.236.221      # Ubuntu instance 2
```

### Alternative: Ansible-Based Setup

You can also automate the SSH key setup using Ansible:

```yaml
---
- hosts: all
  become: true
  tasks:
    - name: Add SSH public key to authorized_keys
      ansible.posix.authorized_key:
        user: "{{ ansible_user }}"
        key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
        state: present
```

---

## Task 3: Conditional Shutdown Automation

### The Challenge

Create an automation that shuts down **only Ubuntu instances**, leaving Amazon Linux instances running. This demonstrates real-world operational scenarios where different server types require different management policies.

### Understanding Ansible Facts

Ansible automatically gathers system information (facts) about target hosts:

```yaml
# View all gathered facts
- name: Display all facts
  ansible.builtin.debug:
    var: ansible_facts
```

**Key OS-related facts:**
- `ansible_facts['os_family']` - OS family (RedHat, Debian, etc.)
- `ansible_facts['distribution']` - Specific distribution
- `ansible_facts['distribution_version']` - Version number

### Conditional Shutdown Playbook

```yaml
---
- hosts: all
  become: true

  tasks:
    - name: Shutdown ubuntu instances only
      ansible.builtin.command: /sbin/shutdown -t now
      when: ansible_facts['os_family'] == "RedHat"
```

### Understanding the Condition

```yaml
when: ansible_facts['os_family'] == "RedHat"
```

**Wait, why RedHat for Ubuntu?** This is a common point of confusion:
- Ubuntu belongs to the **Debian** family
- Amazon Linux belongs to the **RedHat** family
- The condition should be `== "Debian"` for Ubuntu instances

### Corrected Playbook

```yaml
---
- hosts: all
  become: true

  tasks:
    - name: Shutdown ubuntu instances only
      ansible.builtin.command: /sbin/shutdown -t now
      when: ansible_facts['os_family'] == "Debian"
```

### Execution and Verification

```bash
# Run the shutdown playbook
ansible-playbook -i inventory.ini ec2_shutdown.yaml --vault-password-file vault.pass
```

**Expected Output:**
```
PLAY [all] *********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [ec2-user@52.201.113.106]
ok: [ubuntu@35.168.17.47]
ok: [ubuntu@54.158.236.221]

TASK [Shutdown ubuntu instances only] ******************************************
skipping: [ec2-user@52.201.113.106]  # Amazon Linux - skipped
changed: [ubuntu@35.168.17.47]       # Ubuntu - shutdown initiated
changed: [ubuntu@54.158.236.221]     # Ubuntu - shutdown initiated

PLAY RECAP *********************************************************************
ec2-user@52.201.113.106    : ok=1    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
ubuntu@35.168.17.47        : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
ubuntu@54.158.236.221      : ok=1    changed=1    unreachable=0    failed=0    rescued=0    ignored=0
```

---

## Error Handling and Best Practices

### Advanced Error Handling

The project includes sophisticated error handling examples:

```yaml
---
- hosts: webservers
  become: true

  tasks:
    - name: Install security updates
      ansible.builtin.apt:
        name: "{{ item }}"
        state: latest
      loop:
        - openssl
        - openssh
      ignore_errors: yes  # Continue even if this task fails

    - name: Check if docker is installed
      ansible.builtin.command: docker --version
      register: output
      ignore_errors: yes

    - name: Display docker version check result
      ansible.builtin.debug:
        var: output

    - name: Install docker
      ansible.builtin.apt:
        name: docker.io
        state: present
      when: output.failed  # Only install if docker check failed
```

### Key Error Handling Techniques

#### 1. `ignore_errors: yes`
```yaml
- name: Risky operation
  ansible.builtin.command: some-risky-command
  ignore_errors: yes
```

#### 2. `failed_when` Conditions
```yaml
- name: Check if file exists and fail if it does
  ansible.builtin.command: ls /tmp/this_should_not_be_here
  register: result
  failed_when:
    - result.rc == 0
    - '"No such" not in result.stderr'
```

#### 3. Conditional Execution with `when`
```yaml
- name: Install package only if not present
  ansible.builtin.apt:
    name: docker.io
    state: present
  when: output.failed
```

---

## Security Considerations

### Ansible Vault Implementation

#### 1. Create Vault Password File
```bash
# Create base64 encoded password
echo "your-secure-password" | base64 > vault.pass
```

#### 2. Encrypt Sensitive Data
```bash
# Create encrypted variables file
ansible-vault create group_vars/all/pass.yml --vault-password-file vault.pass
```

#### 3. Store AWS Credentials Securely
```yaml
# In group_vars/all/pass.yml (encrypted)
ec2_access_key: "AKIAIOSFODNN7EXAMPLE"
ec2_secret_key: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

#### 4. Use Vault in Playbooks
```yaml
aws_access_key: "{{ec2_access_key}}"
aws_secret_key: "{{ec2_secret_key}}"
```

### Best Security Practices

1. **Never commit unencrypted secrets**
2. **Use least privilege IAM policies**
3. **Rotate access keys regularly**
4. **Use separate AWS accounts for different environments**
5. **Enable CloudTrail for audit logging**

---

## Advanced Features

### Dynamic Inventory

Instead of static inventory files, use dynamic inventory for AWS:

```yaml
# aws_ec2.yml
plugin: aws_ec2
regions:
  - us-east-1
filters:
  tag:Environment: production
keyed_groups:
  - key: tags.Environment
    prefix: env
  - key: instance_type
    prefix: type
```

### Tag-Based Management

```yaml
- name: Shutdown instances by tag
  amazon.aws.ec2_instance:
    instance_ids: "{{ item.id }}"
    state: stopped
  loop: "{{ ec2_instances }}"
  when: item.tags.Environment == "development"
```

### Multi-Region Deployment

```yaml
- name: Create instances in multiple regions
  amazon.aws.ec2_instance:
    name: "{{ item.name }}"
    region: "{{ item.region }}"
    # ... other parameters
  loop:
    - { name: "web-1", region: "us-east-1" }
    - { name: "web-2", region: "us-west-2" }
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. "No such file or directory" for .pem file
```bash
# Solution: Check file path and permissions
ls -la key-p3.pem
chmod 400 key-p3.pem
```

#### 2. "Permission denied (publickey)" SSH errors
```bash
# Solution: Verify SSH key setup
ssh -i key-p3.pem -v ec2-user@<IP>
```

#### 3. "Invalid AMI ID" errors
```bash
# Solution: Verify AMI ID exists in your region
aws ec2 describe-images --image-ids ami-052064a798f08f0d3
```

#### 4. Vault decryption errors
```bash
# Solution: Verify vault password file
ansible-vault view group_vars/all/pass.yml --vault-password-file vault.pass
```

### Debugging Techniques

#### 1. Verbose Output
```bash
ansible-playbook playbook.yaml -vvv
```

#### 2. Check Mode (Dry Run)
```bash
ansible-playbook playbook.yaml --check
```

#### 3. Step-by-Step Execution
```bash
ansible-playbook playbook.yaml --step
```

#### 4. Debug Variables
```yaml
- name: Debug ansible facts
  ansible.builtin.debug:
    var: ansible_facts['os_family']
```

---

## Conclusion

This Ansible EC2 automation project demonstrates essential concepts for infrastructure automation:

### Key Takeaways

1. **Idempotency** is crucial for safe automation
2. **Loops** eliminate repetitive code and improve maintainability
3. **Conditionals** enable intelligent decision-making in automation
4. **Security** through Ansible Vault protects sensitive data
5. **Error handling** makes automation robust and production-ready

### Real-World Applications

This project pattern can be extended for:
- **Multi-environment deployments** (dev, staging, production)
- **Auto-scaling group management**
- **Disaster recovery automation**
- **Cost optimization** through scheduled shutdowns
- **Security patching** across different OS families

### Next Steps

1. **Implement monitoring** with CloudWatch integration
2. **Add backup automation** for critical instances
3. **Create rollback procedures** for failed deployments
4. **Implement CI/CD pipelines** for playbook testing
5. **Add compliance checks** for security standards

### Resources for Further Learning

- [Ansible Documentation](https://docs.ansible.com/)
- [AWS Ansible Collection](https://docs.ansible.com/ansible/latest/collections/amazon/aws/)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
- [Infrastructure as Code Patterns](https://martinfowler.com/bliki/InfrastructureAsCode.html)

---

*This guide provides a comprehensive foundation for Ansible EC2 automation. The concepts and patterns demonstrated here are directly applicable to production environments and can be extended to manage complex, multi-tier infrastructure deployments.*

**Happy Automating! 🚀**
