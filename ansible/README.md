# n7m Ansible Infrastructure

This directory contains Ansible playbooks and automation for deploying the n7m (Nominatim) geocoding service to AWS.

## Prerequisites

- Python 3.x with `uv` package manager
- AWS CLI configured with appropriate credentials
- An AWS account with permissions for EC2, VPC, ECS, RDS, EFS, Route 53, ACM, IAM, and Secrets Manager
- An EC2 key pair in your target region
- A domain managed in Route 53 with an ACM certificate

## Quick Start

```bash
# Set AWS credentials
export AWS_ACCESS_KEY_ID=your-access-key
export AWS_SECRET_ACCESS_KEY=your-secret-key

# Create your inventory from the example
cp -r inventories/example inventories/eu-west-1-dev

# Edit the inventory files with your configuration
# See inventories/README.md for details

# Initialize the virtual environment and bring up infrastructure
make up AWS_REGION=eu-west-1-dev
```

## Make Targets

Run `make help` to see all available targets:

| Target | Description |
|--------|-------------|
| `make help` | Show available targets and usage information |
| `make up` | Bring up the complete infrastructure stack |
| `make down` | Tear down all infrastructure (destructive!) |
| `make playbook PLAYBOOK=<name>` | Run a specific playbook from `playbooks/` |
| `make reset` | Reset the n7m database (re-runs feed with reset) |
| `make venv` | Initialize or update the Python virtual environment |
| `make ssh` | SSH to the bastion server |
| `make exec TARGET=<container>` | Execute a shell in an ECS container |

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `AWS_REGION` | `eu-west-1` | The inventory directory to use (e.g., `eu-west-1-dev`) |
| `AWS_SSH_KEY` | (required for ssh) | Path to the SSH private key for bastion access |
| `ANSIBLE_EXTRA_VARS` | `''` | Additional variables to pass to ansible-playbook |
| `ANSIBLE_VERBOSITY` | `0` | Debug verbosity level (0-5) |

### Examples

```bash
# Bring up infrastructure in a different region
make up AWS_REGION=us-east-1-prod

# Run only the ECS playbook
make playbook PLAYBOOK=ecs-up AWS_REGION=eu-west-1-dev

# SSH to the bastion
AWS_SSH_KEY=~/.ssh/my-keypair.pem make ssh

# Execute shell in the web container
make exec TARGET=web

# Execute shell in the API container
make exec TARGET=api

# Tear down with verbose output
make down AWS_REGION=eu-west-1-dev ANSIBLE_VERBOSITY=2
```

## Architecture Overview

The playbooks deploy the following AWS infrastructure:

```
                                    ┌─────────────────────────────────────────────┐
                                    │                   VPC                       │
    ┌──────────┐                    │  ┌─────────────────┐  ┌─────────────────┐  │
    │ Internet │◄───────────────────┼──│  Public Subnet  │  │  Public Subnet  │  │
    └──────────┘                    │  │  (AZ-a)         │  │  (AZ-b)         │  │
         │                          │  │  ┌───────────┐  │  └─────────────────┘  │
         │                          │  │  │  Bastion  │  │                       │
         ▼                          │  │  └───────────┘  │                       │
    ┌─────────┐                     │  │  ┌───────────┐  │                       │
    │   ALB   │◄────────────────────┼──┼──│    NAT    │  │                       │
    └─────────┘                     │  │  └───────────┘  │                       │
         │                          │  └─────────────────┘                       │
         │                          │                                            │
         ▼                          │  ┌─────────────────┐  ┌─────────────────┐  │
    ┌─────────────────────┐         │  │ Private Subnet  │  │ Private Subnet  │  │
    │    ECS Cluster      │         │  │  (AZ-a)         │  │  (AZ-b)         │  │
    │  ┌───┐ ┌───┐ ┌───┐  │         │  │                 │  │                 │  │
    │  │web│ │api│ │ui │  │◄────────┼──│  ECS Tasks      │  │                 │  │
    │  └───┘ └───┘ └───┘  │         │  │  EFS Mount      │  │                 │  │
    │       ┌───┐         │         │  │                 │  │                 │  │
    │       │mcp│         │         │  └────────┬────────┘  └────────┬────────┘  │
    │       └───┘         │         │           │                    │           │
    └─────────────────────┘         │           ▼                    ▼           │
                                    │  ┌─────────────────────────────────────┐   │
                                    │  │         Aurora PostgreSQL           │   │
                                    │  │           (PostGIS)                 │   │
                                    │  └─────────────────────────────────────┘   │
                                    │                                            │
                                    │  ┌─────────────────────────────────────┐   │
                                    │  │              EFS                    │   │
                                    │  │         (OSM Data Store)            │   │
                                    │  └─────────────────────────────────────┘   │
                                    └────────────────────────────────────────────┘
```

### Components

- **VPC**: Isolated network with public and private subnets across multiple AZs
- **Bastion Host**: SSH jump server for administration and running Ansible tasks within the VPC
- **Application Load Balancer (ALB)**: HTTPS termination and traffic distribution
- **ECS Cluster (Fargate)**: Container orchestration with Service Connect for service discovery
- **Aurora PostgreSQL**: Managed database with PostGIS extensions for geocoding
- **EFS**: Shared file storage for OpenStreetMap data files
- **Route 53**: DNS management for the application domain

### ECS Services

| Service | Port | Description |
|---------|------|-------------|
| `web` | 80 | Nginx reverse proxy (public via ALB) |
| `api` | 8080 | Nominatim API server |
| `ui` | 80 | Web UI for geocoding |
| `mcp` | 8000 | Model Context Protocol server |

## Playbooks

### Main Playbooks

| Playbook | Description |
|----------|-------------|
| `up.yaml` | Orchestrates bringing up the complete infrastructure |
| `down.yaml` | Orchestrates tearing down all infrastructure |

### Component Playbooks (in `playbooks/`)

Run individual playbooks with `make playbook PLAYBOOK=<name>`:

| Playbook | Description |
|----------|-------------|
| `vpc-up.yaml` | Creates VPC, subnets, route tables, NAT gateway |
| `vpc-down.yaml` | Tears down VPC and networking |
| `security-up.yaml` | Creates security groups for all components |
| `security-down.yaml` | Removes security groups |
| `bastion-up.yaml` | Launches and configures bastion EC2 instance |
| `bastion-down.yaml` | Terminates bastion instance |
| `efs-up.yaml` | Creates EFS file system and downloads OSM data |
| `efs-down.yaml` | Removes EFS and mount targets |
| `postgis-up.yaml` | Creates Aurora PostgreSQL cluster with PostGIS |
| `postgis-down.yaml` | Removes RDS cluster and related resources |
| `ecs-up.yaml` | Creates ECS cluster, services, ALB, and Route 53 records |
| `ecs-down.yaml` | Removes ECS services, ALB, and DNS records |

## Custom Modules

This project includes custom Ansible modules in `library/`:

| Module | Description |
|--------|-------------|
| `cidr_allocate` | Allocates available CIDR blocks from a master range using best-fit algorithm |
| `cloudmap_namespace` | Creates/deletes AWS Cloud Map namespaces for ECS Service Connect |
| `cloudmap_info` | Retrieves information about Cloud Map namespaces |
| `ecs_service` | Enhanced ECS service management with Service Connect support |
| `ecs_task` | Run ECS tasks (used for feed operations) |

## Directory Structure

```
ansible/
├── Makefile              # Build targets for common operations
├── README.md             # This file
├── requirements.txt      # Python dependencies
├── up.yaml               # Main playbook to bring up infrastructure
├── down.yaml             # Main playbook to tear down infrastructure
├── main.yaml             # Container orchestration (legacy)
├── inventories/          # Inventory configurations
│   ├── README.md         # Inventory documentation
│   └── example/          # Example inventory to copy
├── playbooks/            # Component-specific playbooks
│   ├── vpc-up.yaml
│   ├── vpc-down.yaml
│   ├── security-up.yaml
│   ├── security-down.yaml
│   ├── bastion-up.yaml
│   ├── bastion-down.yaml
│   ├── efs-up.yaml
│   ├── efs-down.yaml
│   ├── postgis-up.yaml
│   ├── postgis-down.yaml
│   ├── ecs-up.yaml
│   └── ecs-down.yaml
├── tasks/                # Reusable task files
│   ├── services-up.yaml
│   └── services-down.yaml
└── library/              # Custom Ansible modules
    ├── cidr_allocate.py
    ├── cloudmap_namespace.py
    ├── cloudmap_info.py
    ├── ecs_service.py
    └── ecs_task.py
```

## Configuration

See [inventories/README.md](inventories/README.md) for detailed configuration documentation.

### Key Variables

| Variable | File | Description |
|----------|------|-------------|
| `prefix_v` | host_vars/localhost.yaml | Resource naming prefix |
| `region_v` | host_vars/localhost.yaml | AWS region |
| `keypair_v` | host_vars/localhost.yaml | EC2 key pair name |
| `bastion_ami_v` | host_vars/localhost.yaml | Ubuntu AMI for bastion |
| `osm_continent_v` | group_vars/_bastion.yaml | Geofabrik continent |
| `osm_region_v` | group_vars/_bastion.yaml | Geofabrik region for OSM data |
| `n7m_domain` | group_vars/_bastion.yaml | Application domain |
| `ca_certificate_arn` | group_vars/_bastion.yaml | ACM certificate ARN |
| `docker_credentials_arn` | group_vars/_bastion.yaml | Docker Hub credentials secret |

## Troubleshooting

### Increase Verbosity

```bash
make up ANSIBLE_VERBOSITY=2
# or
ANSIBLE_VERBOSITY=3 make playbook PLAYBOOK=ecs-up
```

### Check Running Services

```bash
# List ECS tasks
aws ecs list-tasks --cluster n7m-eu-west-1-dev-ecs-cluster

# View service logs
aws logs tail /ecs/n7m --follow
```

### Connect to Containers

```bash
# Interactive shell in web container
make exec TARGET=web

# Interactive shell in API container
make exec TARGET=api
```

### Common Issues

1. **"Could not find running bastion server"**: Ensure the bastion is running with `make playbook PLAYBOOK=bastion-up`

2. **"Error: AWS_SSH_KEY environment variable is required"**: Set the path to your SSH key: `AWS_SSH_KEY=~/.ssh/key.pem make ssh`

3. **Timeout during ECS service creation**: Services may take several minutes to stabilize. Check CloudWatch logs for container startup issues.

4. **Database connection failures**: Ensure security groups allow traffic between ECS tasks and Aurora. Check that PostGIS extensions were installed.

## Documentation Links

- [Ansible Documentation](https://docs.ansible.com/)
- [AWS EC2 Dynamic Inventory](https://docs.ansible.com/ansible/latest/collections/amazon/aws/aws_ec2_inventory.html)
- [Amazon ECS Documentation](https://docs.aws.amazon.com/ecs/)
- [Aurora PostgreSQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraPostgreSQL.html)
- [Geofabrik Downloads](https://download.geofabrik.de/)
