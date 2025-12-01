# Ansible Inventories

This directory contains Ansible inventory configurations for deploying n7m infrastructure to AWS.

## Documentation

- **Ansible Inventory Documentation**: https://docs.ansible.com/ansible/latest/inventory_guide/index.html
- **AWS EC2 Dynamic Inventory Plugin**: https://docs.ansible.com/ansible/latest/collections/amazon/aws/aws_ec2_inventory.html
- **Ansible Variable Precedence**: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable

## Directory Naming Convention

Inventory directories should follow the naming pattern:

```
<aws-region>[-<environment>]
```

**Examples:**
- `eu-west-1` - Production deployment in EU West (Ireland)
- `eu-west-1-dev` - Development deployment in EU West
- `us-east-1-staging` - Staging deployment in US East (N. Virginia)
- `ap-southeast-1` - Production deployment in Asia Pacific (Singapore)

This naming convention:
1. Clearly identifies the target AWS region at a glance
2. Allows multiple environments per region (dev, staging, prod)
3. Matches the `region_v` variable used in playbooks
4. Enables easy command-line selection: `ansible-playbook -i inventories/eu-west-1-dev up.yaml`

## Directory Structure

Each inventory directory should contain:

```
inventories/
├── README.md                    # This file
├── example/                     # Example configuration (copy this to start)
│   ├── aws_ec2.yaml            # Dynamic inventory for EC2 discovery
│   ├── localhost.yaml          # Static inventory for local tasks
│   ├── group_vars/
│   │   ├── all.yaml            # Shared variables for all hosts
│   │   └── _bastion.yaml       # Variables specific to bastion hosts
│   └── host_vars/
│       └── localhost.yaml      # Variables specific to localhost
└── eu-west-1-dev/              # Your actual deployment (gitignored)
    ├── aws_ec2.yaml
    ├── localhost.yaml
    ├── group_vars/
    │   ├── all.yaml
    │   └── _bastion.yaml
    └── host_vars/
        └── localhost.yaml
```

## Getting Started

1. **Copy the example directory** to create your deployment configuration:
   ```bash
   cp -r ansible/inventories/example ansible/inventories/eu-west-1-dev
   ```

2. **Edit the configuration files** with your specific values:
   - Update AWS region, keypair name, and AMI IDs
   - Set your Docker credentials ARN and KMS key
   - Configure your domain and ACM certificate
   - Choose your OSM region for geocoding data

3. **Set AWS credentials** as environment variables:
   ```bash
   export AWS_ACCESS_KEY_ID=your-access-key
   export AWS_SECRET_ACCESS_KEY=your-secret-key
   ```

4. **Run the playbooks**:
   ```bash
   cd ansible
   ansible-playbook -i inventories/eu-west-1-dev up.yaml
   ```

## Variable Files Reference

### `group_vars/all.yaml`
Shared variables that apply to all hosts (localhost and bastion):
- `prefix_v` - Resource naming prefix (e.g., `n7m-eu-west-1-dev`)
- `region_v` - AWS region (e.g., `eu-west-1`)
- `keypair_v` - EC2 key pair name
- `owner_v` - Resource owner for tagging

### `host_vars/localhost.yaml`
Variables specific to localhost for AWS infrastructure provisioning:
- `instance_type_v` - Bastion instance type (e.g., `t4g.micro`)
- `bastion_ami_v` - Ubuntu AMI ID for bastion host

### `group_vars/_bastion.yaml`
Variables specific to bastion hosts running AWS operations from within the VPC:
- SSH connection settings (`ansible_user`, `ansible_ssh_private_key_file`)
- AWS credentials (`access_key_v`, `secret_key_v`)
- OSM data configuration (`osm_continent_v`, `osm_region_v`)
- Secrets Manager ARNs (`docker_credentials_arn`, `secretsmanager_key`)
- Application domain settings (`n7m_domain`, `ca_certificate_arn`)

### `aws_ec2.yaml`
Dynamic inventory plugin configuration for discovering EC2 instances:
- Scans specified region for running instances
- Groups instances by their `Role` tag
- Sets connection parameters from instance attributes

### `localhost.yaml`
Static inventory defining localhost for local task execution.

## Security Notes

- **Never commit credentials** to version control
- Inventory directories (except `example/`) are gitignored by default
- AWS credentials are read from environment variables
- Secrets (passwords, tokens) are stored in AWS Secrets Manager
