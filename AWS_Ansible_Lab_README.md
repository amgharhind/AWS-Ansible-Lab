# AWS-Ansible-Lab — EC2 Instance Management with Ansible

## Overview

This project provides a collection of **Ansible playbooks** to manage the full lifecycle of AWS EC2 instances — from creation to termination. Using the `amazon.aws` Ansible collection, each playbook automates a specific operation (create, start, stop, tag, terminate) on EC2 instances in a reproducible and consistent way. AWS credentials are handled securely through environment variables, and all playbooks run locally against the AWS API without requiring a remote inventory.

## Motivation and Importance

Manually managing EC2 instances through the AWS Console or CLI is repetitive and error-prone at scale. Automating these operations with Ansible offers:

- **Reproducibility:** Every instance is launched with the same configuration every time, eliminating human error.
- **Security:** AWS credentials are never hardcoded — they are injected at runtime via environment variables.
- **Auditability:** Infrastructure changes are version-controlled as YAML playbooks, making them reviewable and trackable.
- **Speed:** Common operations (start, stop, terminate) are reduced to a single command.

## Project Structure

```plaintext
AWS-Ansible-Lab/
├── ansible.cfg                              # Ansible configuration (inventory, Python path, privilege escalation)
├── hosts.ini                                # Inventory file (localhost for local execution)
├── create_ec2_instance.yml                  # Creates a new EC2 instance
├── create_instances_hosts_group.yml         # Creates an instance, adds it to a host group, and waits for SSH
├── start_ec2_instance.yml                   # Starts an existing stopped EC2 instance
├── Start_existing_EC2_instance_exemple.yml  # Example playbook to start an instance (template with placeholders)
├── stop_ec2_instance.yml                    # Stops a running EC2 instance
├── tag_ec2_instance.yml                     # Adds tags to an existing EC2 instance
└── terminate_ec2_instance.yml               # Permanently terminates an EC2 instance
```

## Prerequisites

- **Python 3.8+** and a virtual environment set up at `/home/ubuntu/ansible_ec2/.env/`
- **Ansible** installed inside the virtual environment
- **amazon.aws** Ansible collection installed:
  ```bash
  ansible-galaxy collection install amazon.aws
  ```
- **boto3** and **botocore** Python libraries:
  ```bash
  pip install boto3 botocore
  ```
- An **AWS account** with an IAM user that has EC2 permissions, and a key pair named `key_pub_ec2` created in the `eu-north-1` region.

## Configuration

### `ansible.cfg`

Defines global Ansible settings for this project:

```ini
[defaults]
collections_paths = /home/ubuntu/ansible_ec2/.env/lib/python3.8/site-packages/ansible_collections
inventory = hosts
interpreter_python = /home/ubuntu/ansible_ec2/.env/bin/python

[privilege_escalation]
become = True
become_method = sudo
```

### `hosts.ini`

The inventory file points all playbooks to run locally (against the AWS API, not a remote host):

```ini
[local]
localhost ansible_connection=local ansible_python_interpreter=/home/ubuntu/ansible_ec2/.env/bin/python
```

### AWS Credentials

Credentials are **never hardcoded**. Before running any playbook, export your AWS credentials as environment variables:

```bash
export AWS_ACCESS_KEY=your_access_key_here
export AWS_SECRET_KEY=your_secret_key_here
```

All playbooks retrieve them at runtime using Ansible's `lookup('env', ...)` function.

## Playbooks

### 1. Create an EC2 Instance (`create_ec2_instance.yml`)

Launches a new EC2 instance with the following configuration:

| Parameter | Value |
|---|---|
| Instance Type | `t3.nano` |
| AMI | `ami-0506d6d51f1916a96` |
| Region | `eu-north-1` |
| Key Pair | `key_pub_ec2` |
| Tag | `Name: Ansible_EC2` |

```bash
ansible-playbook create_ec2_instance.yml
```

---

### 2. Create Instance + Add to Host Group (`create_instances_hosts_group.yml`)

An extended creation playbook that goes further than a simple launch — it:

1. **Launches** a new EC2 instance with the same parameters as above.
2. **Registers** the result and adds the instance's public IP to the `launched` host group dynamically.
3. **Waits for SSH** to become available on port 22 (with a 60-second delay and 320-second timeout).
4. **Verifies** the instance is in `running` state using `amazon.aws.ec2_instance_info`.
5. **Prints** instance details for debugging.

```bash
ansible-playbook create_instances_hosts_group.yml
```

---

### 3. Start an Existing Instance (`start_ec2_instance.yml`)

Starts a previously stopped EC2 instance by its instance ID. Update the `instance_id` variable before running.

```bash
ansible-playbook start_ec2_instance.yml
```

> A template version with placeholder values is provided in `Start_existing_EC2_instance_exemple.yml` for reference.

---

### 4. Stop an Instance (`stop_ec2_instance.yml`)

Gracefully stops a running EC2 instance (the instance is stopped but not terminated — no data is lost). Update the `instance_id` variable before running.

```bash
ansible-playbook stop_ec2_instance.yml
```

---

### 5. Tag an Instance (`tag_ec2_instance.yml`)

Adds metadata tags to an existing EC2 instance. Default tags applied:

| Key | Value |
|---|---|
| `Environment` | `Development` |
| `Department` | `IT` |

Replace the `resource` field with your actual instance ID before running.

```bash
ansible-playbook tag_ec2_instance.yml
```

---

### 6. Terminate an Instance (`terminate_ec2_instance.yml`)

**Permanently destroys** an EC2 instance (state: `absent`). This action is irreversible. Update the `instance_id` variable before running.

```bash
ansible-playbook terminate_ec2_instance.yml
```

>  **Warning:** Termination permanently deletes the instance and its associated storage. Make sure you are targeting the correct instance ID.

## Technology Stack

- **Automation:** Ansible
- **Cloud Provider:** AWS (Amazon EC2, `eu-north-1` region)
- **Ansible Collection:** `amazon.aws`
- **Language:** Python 3.8 (via virtual environment)
- **Libraries:** `boto3`, `botocore`
- **Credential Management:** Environment variables (`AWS_ACCESS_KEY`, `AWS_SECRET_KEY`)

## Limitations and Future Work

- **Single Region:** All playbooks are hardcoded for the `eu-north-1` region. Parameterizing the region via `--extra-vars` would make them more flexible.
- **Static Instance IDs:** Start, stop, tag, and terminate playbooks require manually updating the `instance_id` variable. A future improvement could use dynamic inventory to look up instances by tag name instead.
- **No Security Group Management:** The playbooks do not configure security groups or VPC settings. Adding these would make the setup more complete for real deployments.
- **No Rollback:** If a playbook fails mid-execution, there is no automatic rollback. Adding error handling with `block/rescue` would improve reliability.
