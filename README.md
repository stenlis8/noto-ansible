# WordPress and MariaDB Deployment with Ansible

## Overview

This repository contains the Ansible configuration used to deploy a two-tier WordPress platform on AWS EC2 instances.

The infrastructure is provisioned separately with Terraform. This Ansible project is responsible only for the configuration and application deployment layer.

The deployment installs and configures:

* Docker
* Docker Compose
* WordPress application container
* MariaDB database container
* SELinux permissions for Docker bind mounts
* Environment files generated from AWS Secrets Manager credentials

## Architecture

The platform consists of two EC2 instances:

```text
Application EC2
└── Docker Compose
    └── WordPress container

Database EC2
└── Docker Compose
    └── MariaDB container
```

The WordPress container connects to the MariaDB container running on the separate database EC2 instance.

Both instances are expected to be private EC2 instances and accessed through AWS Systems Manager Session Manager.

## Repository Structure

```text
.
├── inventory.ini
├── playbook.yml
├── README.md
├── requirements.yml
└── roles
    └── wp_mariadb_stack
        ├── defaults
        │   └── main.yml
        ├── tasks
        │   ├── docker.yml
        │   ├── main.yml
        │   ├── mariadb.yml
        │   ├── secrets.yml
        │   ├── selinux.yml
        │   └── wordpress.yml
        └── templates
            ├── compose.yml.j2
            ├── mariadb-compose.yml.j2
            ├── mariadb.env.j2
            ├── wordpress-compose.yml.j2
            └── wordpress.env.j2
```

## Role Responsibilities

### `docker.yml`

Installs Docker and Docker Compose plugin on the target EC2 instances.

### `secrets.yml`

Reads database credentials from AWS Secrets Manager.

The secret is expected to contain values similar to:

```json
{
  "root_password": "example-root-password",
  "db_name": "wordpress",
  "username": "wordpress",
  "password": "example-user-password"
}
```

### `selinux.yml`

Checks SELinux status and applies the required SELinux context for Docker bind mounts.

### `wordpress.yml`

Deploys the WordPress Docker Compose stack on the application EC2 instance.

### `mariadb.yml`

Deploys the MariaDB Docker Compose stack on the database EC2 instance.

## Requirements

Install the required Ansible collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

Install required Python packages:

```bash
pip install boto3 botocore
```

If using AWS SSM instead of SSH, install the Session Manager Plugin locally:

```bash
session-manager-plugin --version
```
## Variable file
This file needs to be renamed or created and filed as shown:

```bash
roles/wp_mariadb_stack/defaults/main.yml_example
```
## Inventory

The inventory uses EC2 instance IDs when connecting through AWS SSM.

Example:

```ini
[wordpress]
i-xxxxxxxxxxxxxxxxx

[database]
i-yyyyyyyyyyyyyyyyy

[aws_ec2:children]
wordpress
database

[aws_ec2:vars]
ansible_connection=amazon.aws.aws_ssm
ansible_aws_ssm_region=eu-central-1
ansible_aws_ssm_bucket_name=<ansible-ssm-bucket-name>
ansible_user=ssm-user
ansible_python_interpreter=/usr/bin/python3
ansible_become=true
```

The instance IDs and SSM bucket name are provided by the Terraform outputs.

## Playbook

The main playbook is:

```yaml
- name: Deploy WordPress and MariaDB
  hosts: aws_ec2
  become: true
  roles:
    - wp_mariadb_stack
```

## Usage

Test AWS identity:

```bash
aws sts get-caller-identity
```

Test SSM connection manually:

```bash
aws ssm start-session \
  --target i-xxxxxxxxxxxxxxxxx \
  --region eu-central-1
```

Test Ansible connectivity:

```bash
ansible -i inventory.ini aws_ec2 -m ping
```

Run the deployment:

```bash
ansible-playbook -i inventory.ini playbook.yml
```

## Expected Result

After the playbook finishes:

* Docker is installed on both EC2 instances
* WordPress container is running on the application server
* MariaDB container is running on the database server
* WordPress can connect to MariaDB
* The application is reachable through the AWS Load Balancer URL created by Terraform

## Useful Commands

Check running containers on the WordPress host:

```bash
docker ps
```

Check running containers on the MariaDB host:

```bash
docker ps
```

View WordPress logs:

```bash
docker logs wordpress
```

View MariaDB logs:

```bash
docker logs mariadb
```

Check Docker Compose stack:

```bash
docker compose ps
```

## Notes

* This repository does not create AWS infrastructure.
* Infrastructure is managed by Terraform in a separate repository.
* Credentials are not stored in this repository.
* Database credentials are read from AWS Secrets Manager.
* Generated `.env` files on the servers should be treated as sensitive.
* EC2 instances are expected to have IAM permissions for SSM access.
# noto-ansible
