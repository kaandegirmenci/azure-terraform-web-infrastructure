# Azure Web Server Infrastructure with Terraform

End-to-end infrastructure for a Linux web server on Microsoft Azure, fully provisioned with Terraform. This project builds an entire environment from scratch — networking, security, and compute — and serves an nginx web page, all defined as code in a single configuration.

## What This Project Does

Running `terraform apply` creates 8 Azure resources that together form a working, internet-accessible web server:

- **Resource Group** — logical container for all resources
- **Virtual Network** (10.0.0.0/16) — private network
- **Subnet** (10.0.1.0/24) — web tier segment
- **Network Security Group** — firewall allowing SSH (22) and HTTP (80)
- **Subnet–NSG Association** — applies the firewall rules to the subnet
- **Public IP** (static) — for external access
- **Network Interface** — connects the VM to the network
- **Linux Virtual Machine** — Ubuntu 24.04, running nginx

## Architecture

```
Resource Group (rg-terraform-web)
│
├── Virtual Network (vnet-web) — 10.0.0.0/16
│   └── Subnet (subnet-web) — 10.0.1.0/24
│         │
│         └── associated with ──► Network Security Group (nsg-web)
│                                   ├── allow SSH  (port 22)
│                                   └── allow HTTP (port 80)
│
├── Public IP (static) ──┐
│                         │
└── Network Interface ────┴──► Linux VM (Ubuntu 24.04 + nginx)
```

## Tech Stack

- **Terraform** (IaC)
- **Microsoft Azure** (azurerm provider)
- **Ubuntu 24.04 LTS**
- **nginx**

## Prerequisites

- An Azure account and an active subscription
- Terraform installed
- Azure CLI (authenticated with `az login`)
- An SSH key pair at `~/.ssh/id_rsa.pub`

## How to Use

```bash
# Initialize the working directory
terraform init

# Preview the changes
terraform plan

# Create the infrastructure
terraform apply

# When finished, destroy everything to avoid charges
terraform destroy
```

After `apply`, retrieve the VM's public IP, SSH into it, and install nginx:

```bash
ssh azureuser@<public-ip>
sudo apt update && sudo apt install nginx -y
```

Then open `http://<public-ip>` in a browser to see the nginx welcome page.

## Challenges & Lessons Learned

This project was built hands-on, and real infrastructure rarely works on the first try. A few of the problems I ran into and how I solved them:

- **VM SKU capacity restrictions.** Several regions and VM sizes returned `SkuNotAvailable`. I learned to query available sizes with `az vm list-skus` and settled on a region/size combination (`westus2` + `Standard_B2as_v2`) that had capacity.
- **Region consistency.** Changing the region for one resource broke dependent resources. Lesson: all connected resources (VNet, subnet, NSG, IP, NIC, VM) must share a consistent region, and changing it mid-deployment requires a clean rebuild.
- **State lock and provider errors.** I hit `state lock` issues and the `Provider produced inconsistent result after apply` bug. I learned to resolve locks, clean up partial state, and that re-running `apply` often resolves eventual-consistency issues.
- **"Resource already exists".** Deleting a resource group asynchronously and re-applying too early caused conflicts. Lesson: wait for full deletion (`az group wait --deleted`) before recreating.

These weren't in any tutorial — working through them is where I actually learned how Terraform and Azure behave.

## Next Steps

- Automate nginx installation via cloud-init (no manual SSH step)
- Parameterize the configuration with variables
- Add remote state storage (Azure Storage backend)
- Extend to multiple VMs behind a load balancer

---

Built as part of my journey from IT support into cloud and infrastructure automation.
