# Azure AKS with Terraform, Ansible & CI/CD

This repository contains infrastructure as code for deploying a Flask application on Azure Kubernetes Service (AKS) using Terraform for infrastructure provisioning, Ansible for application deployment, and GitHub Actions for CI/CD automation.

## Architecture Overview

This infrastructure creates:
- **Azure Kubernetes Service (AKS)**: Managed Kubernetes cluster
- **Resource Group**: Container for all Azure resources
- **System-Assigned Identity**: For AKS cluster authentication
- **Log Analytics Workspace**: Optional monitoring and logging
- **Node Pool**: Configurable VM sizes and auto-scaling
- **Network Profile**: Kubenet plugin with standard load balancer

## Repository Structure

```
azure-aks-terraform-ansible-cicd/
├── .github/
│   └── workflows/
│       └── ci-cd.yml              # GitHub Actions CI/CD pipeline
├── ansible/
│   ├── inventory/
│   │   └── hosts                  # Ansible inventory
│   ├── playbooks/
│   │   └── deploy_app.yml         # Helm deployment playbook
│   └── roles/
│       ├── docker                 # Docker role
│       └── app_deploy             # Application deployment role
├── terraform/
│   ├── main.tf                    # Main infrastructure resources
│   ├── providers.tf               # Provider configuration
│   ├── outputs.tf                 # Output values
│   ├── variables.tf               # Variable definitions
│   └── backend.tf                 # Backend and version configuration
├── helm/
│   └── flask-app/
│       ├── Chart.yaml             # Helm chart metadata
│       ├── values.yaml            # Helm chart values
│       └── templates/
│           └── deployment.yaml    # Kubernetes deployment template
├── app/
│   ├── main.py                    # Flask application
│   └── requirements.txt           # Python dependencies
├── Dockerfile                     # Container image definition
└── README.md                      # This file
```

## Prerequisites

- Azure CLI installed and authenticated
- Terraform installed (version 1.0+)
- Ansible installed
- kubectl installed
- Helm installed (for local testing)
- Docker installed (for local builds)
- Azure subscription with appropriate permissions
- GitHub repository with required secrets configured

## Configuration Details

### Azure Resources Created

1. **Resource Group** (`azurerm_resource_group.rg`)
   - Name: rg-aks (configurable)
   - Location: East US (configurable)

2. **AKS Cluster** (`azurerm_kubernetes_cluster.aks`)
   - Name: flask-aks (configurable)
   - System-assigned managed identity
   - RBAC enabled
   - Kubenet network plugin

3. **Default Node Pool**
   - VM Size: Standard_DS2_v2 (configurable)
   - Node Count: 1 (configurable)
   - Optional auto-scaling support

4. **Log Analytics Workspace** (Optional)
   - For monitoring and logging
   - 30-day retention period
   - Diagnostic settings for AKS

### Backend Configuration

The configuration uses Azure Storage for remote state, defined in `backend.tf`:
- Resource Group: `tfstate-rg`
- Storage Account: `yourtfstatestorage` (update with unique name)
- Container: `tfstate`
- State File: `aks.terraform.tfstate`
- Terraform Version: >= 1.0
- AzureRM Provider Version: ~> 3.0

## Setup Instructions

### 1. Setup Backend Storage

Create the storage account for Terraform state:

```bash
# Create resource group for state
az group create --name tfstate-rg --location "East US"

# Create storage account (replace with unique name)
az storage account create \
  --resource-group tfstate-rg \
  --name yourtfstatestorage \
  --sku Standard_LRS \
  --encryption-services blob

# Create container
az storage container create \
  --name tfstate \
  --account-name yourtfstatestorage
```

### 2. Configure GitHub Secrets

Add these secrets to your GitHub repository:

```
ARM_CLIENT_ID          # Azure Service Principal App ID
ARM_CLIENT_SECRET      # Azure Service Principal Password
ARM_SUBSCRIPTION_ID    # Azure Subscription ID
ARM_TENANT_ID          # Azure Tenant ID
GITHUB_TOKEN           # GitHub Personal Access Token
```

### 3. Update Backend Configuration

Before initializing, update the storage account name in `backend.tf`:

```hcl
# In backend.tf, change this line:
storage_account_name = "youruniquestatestorageaccount"
```

### 4. Deploy Infrastructure

```bash
# Navigate to terraform directory
cd terraform

# Initialize Terraform
terraform init

# Plan the deployment
terraform plan

# Apply the configuration
terraform apply
```

### 5. Configure kubectl

After deployment, configure kubectl to connect to your AKS cluster:

```bash
# Get AKS credentials
az aks get-credentials --resource-group rg-aks --name flask-aks

# Verify connection
kubectl get nodes
```

## Variables

| Variable | Description | Type | Default |
|----------|-------------|------|---------|
| `location` | Azure region | string | East US |
| `resource_group_name` | Resource group name | string | rg-aks |
| `aks_cluster_name` | AKS cluster name | string | flask-aks |
| `dns_prefix` | DNS prefix for AKS | string | flask-aks |
| `node_count` | Number of nodes | number | 1 |
| `vm_size` | VM size for nodes | string | Standard_DS2_v2 |
| `enable_auto_scaling` | Enable auto-scaling | bool | false |
| `min_node_count` | Minimum nodes (auto-scaling) | number | 1 |
| `max_node_count` | Maximum nodes (auto-scaling) | number | 3 |
| `enable_log_analytics` | Enable Log Analytics | bool | false |
| `tags` | Resource tags | map(string) | See variables.tf |

## Outputs

| Output | Description |
|--------|-------------|
| `resource_group_name` | Resource group name |
| `aks_cluster_name` | AKS cluster name |
| `aks_cluster_id` | AKS cluster ID |
| `aks_fqdn` | AKS cluster FQDN |
| `aks_node_resource_group` | Auto-generated node resource group |
| `kube_config_raw` | Raw kubeconfig (sensitive) |
| `client_certificate` | Client certificate (sensitive) |
| `client_key` | Client key (sensitive) |
| `cluster_ca_certificate` | Cluster CA certificate (sensitive) |
| `host` | Kubernetes cluster host (sensitive) |
| `log_analytics_workspace_id` | Log Analytics workspace ID |

## CI/CD Pipeline

The GitHub Actions workflow (`ci-cd.yml`) includes three jobs:

1. **Terraform**: Provisions AKS infrastructure
2. **Build-Push**: Builds Docker image and pushes to GitHub Container Registry
3. **Deploy**: Uses Ansible to deploy Helm charts to AKS

### Pipeline Flow:
```
Code Push → Terraform Deploy → Docker Build → Ansible Deploy → App Running
```

## Application Deployment

The application is deployed using:
- **Helm Charts**: Located in `helm/flask-app/`
- **Ansible Playbooks**: Located in `ansible/playbooks/`
- **Docker Images**: Built from `Dockerfile` and pushed to GHCR

## Local Development

### Test Terraform Locally
```bash
cd terraform
terraform init
terraform plan -var="enable_log_analytics=true"
```

### Build Docker Image Locally
```bash
docker build -t flask-app:local .
docker run -p 5000:5000 flask-app:local
```

### Deploy with Helm Locally
```bash
# Install Helm chart
helm install flask-app ./helm/flask-app

# Upgrade existing deployment
helm upgrade flask-app ./helm/flask-app
```

## Monitoring and Logging

When `enable_log_analytics` is set to `true`:
- Log Analytics workspace is created
- AKS diagnostic settings are configured
- Monitor kube-apiserver, controller-manager, scheduler logs
- Cluster autoscaler logs included
- All metrics collected

## Security Considerations

1. **Managed Identity**: Uses system-assigned identity for AKS
2. **RBAC**: Role-based access control enabled
3. **Network Security**: Kubenet plugin with standard load balancer
4. **Secret Management**: Sensitive outputs marked appropriately
5. **Service Principal**: Used for CI/CD authentication
6. **Container Registry**: GitHub Container Registry for image storage

## Customization

### Environment-Specific Deployments
Create `.tfvars` files for different environments:

```hcl
# dev.tfvars
location = "East US"
node_count = 1
vm_size = "Standard_DS2_v2"
enable_log_analytics = false

# prod.tfvars
location = "West US 2"
node_count = 3
vm_size = "Standard_DS3_v2"
enable_log_analytics = true
enable_auto_scaling = true
```

### Auto-scaling Configuration
```bash
terraform apply -var="enable_auto_scaling=true" -var="min_node_count=1" -var="max_node_count=5"
```

## Troubleshooting

### Common Issues

1. **AKS Cluster Access**: Ensure service principal has Contributor role
2. **Backend Storage**: Verify storage account exists and is accessible
3. **Docker Build**: Check Dockerfile and app dependencies
4. **Helm Deployment**: Verify chart syntax and values
5. **Network Issues**: Check AKS network configuration

### Debug Commands

```bash
# Check AKS cluster status
az aks show --resource-group rg-aks --name flask-aks

# Get AKS credentials
az aks get-credentials --resource-group rg-aks --name flask-aks

# Check pods
kubectl get pods -n default

# Check Helm releases
helm list

# View logs
kubectl logs -f deployment/flask-app
```

## Clean Up

To destroy the infrastructure:

```bash
# Destroy Terraform resources
cd terraform
terraform destroy

# Delete Helm releases (if any)
helm uninstall flask-app
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make changes and test locally
4. Submit a pull request
5. Ensure CI/CD pipeline passes

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Support

For issues and questions:
- Check the troubleshooting section
- Review Azure AKS documentation
- Open an issue in this repository
