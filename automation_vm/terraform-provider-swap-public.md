# Terraform Proxmox Provider Migration: Telmate to BPG

## Summary

This document records the migration of a Proxmox Terraform project from the legacy Telmate provider to the BPG Proxmox provider.

The migration replaced the old Telmate VM resource model with BPG's `proxmox_virtual_environment_vm` resource model, cleaned stale Telmate state, and confirmed Terraform can initialize and validate using only the BPG provider.

## Goal

- Replace Telmate Proxmox provider usage with BPG Proxmox provider usage.
- Convert VM resource syntax from Telmate to BPG.
- Keep Terraform execution under a service account.
- Keep credentials out of committed Terraform code.
- Preserve a clean project structure for future automation work.

## Starting Point

The Terraform project previously used Telmate-style provider and VM configuration.

Old Telmate provider style:

```hcl
provider "proxmox" {
  pm_api_url          = "https://${var.proxmox_host}:8006/api2/json"
  pm_api_token_id     = var.proxmox_token_id
  pm_api_token_secret = var.proxmox_token_secret
  pm_tls_insecure     = true
}
```

Old Telmate VM resource type:

```hcl
resource "proxmox_vm_qemu" "ubuntu_server_01" {
  # VM configuration
}
```

After the initial provider requirement change, Terraform showed a mixed state:

- Configuration required BPG.
- Terraform state still referenced Telmate.
- The VM resource file still needed conversion.

## New Provider Configuration

The provider block was updated to BPG syntax:

```hcl
provider "proxmox" {
  endpoint  = "https://${var.proxmox_host}:8006/"
  api_token = "${var.proxmox_token_id}=${var.proxmox_token_secret}"
  insecure  = true
}
```

SSH was intentionally not added to the provider block during the initial migration because the first VM workflow did not require BPG SSH-based file/snippet operations.

## Provider Requirement

The Terraform provider requirement was updated to use BPG:

```hcl
terraform {
  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "~> 0.104"
    }
  }
}
```

The `hashicorp/local` provider remained in the project as an existing dependency.

## VM Resource Conversion

The VM resource was converted from:

```hcl
resource "proxmox_vm_qemu" "ubuntu_server_01" {
  # Telmate schema
}
```

to:

```hcl
resource "proxmox_virtual_environment_vm" "ubuntu_server_01" {
  # BPG schema
}
```

Major schema changes included:

| Telmate | BPG |
|---|---|
| `target_node` | `node_name` |
| `vmid` | `vm_id` |
| `agent = 1` | `agent { enabled = true }` |
| `qemu_os = "l26"` | `operating_system { type = "l26" }` |
| `memory = 4096` | `memory { dedicated = 4096 }` |
| `balloon = 0` | `memory { floating = 0 }` |
| `scsihw` | `scsi_hardware` |
| `efidisk {}` | `efi_disk {}` |
| `boot = "order=ide2;scsi0"` | `boot_order = ["ide2", "scsi0"]` |
| `disk { slot = "scsi0" }` | `disk { interface = "scsi0" }` |
| `disk { type = "cdrom" }` | `cdrom {}` |
| `network {}` | `network_device {}` |
| `tags = ""` | `tags = []` |

## Converted BPG VM Example

```hcl
resource "proxmox_virtual_environment_vm" "ubuntu_server_01" {
  name        = "ubuntu-server-01"
  node_name   = var.proxmox_node
  vm_id       = 148
  description = "Ubuntu Server VM deployed by Terraform"

  on_boot = false
  started = false

  bios    = "ovmf"
  machine = "q35"

  operating_system {
    type = "l26"
  }

  agent {
    enabled = true
  }

  cpu {
    cores   = 2
    sockets = 1
    type    = "host"
  }

  memory {
    dedicated = 4096
    floating  = 0
  }

  scsi_hardware = "virtio-scsi-single"

  efi_disk {
    datastore_id      = var.vm_storage
    file_format       = "raw"
    type              = "4m"
    pre_enrolled_keys = false
  }

  boot_order = ["ide2", "scsi0"]

  disk {
    datastore_id = var.vm_storage
    interface    = "scsi0"
    size         = 32
    file_format  = "raw"
    discard      = "on"
    iothread     = false
    ssd          = true
  }

  cdrom {
    interface = "ide2"
    file_id   = var.ubuntu_iso
  }

  network_device {
    bridge = "vmbr0"
    model  = "virtio"
  }

  network_device {
    bridge = "vmbr1"
    model  = "virtio"
  }

  tags = []
}
```

## Variable Layout

The project uses `variables.tf` for declarations only.

Example declarations:

```hcl
variable "proxmox_host" {
  description = "Proxmox API host or IP"
  type        = string
}

variable "proxmox_node" {
  description = "Proxmox node name"
  type        = string
}

variable "proxmox_token_id" {
  description = "Proxmox API token ID"
  type        = string
  sensitive   = true
}

variable "proxmox_token_secret" {
  description = "Proxmox API token secret"
  type        = string
  sensitive   = true
}

variable "vm_storage" {
  description = "Proxmox datastore for VM disks"
  type        = string
}

variable "ubuntu_iso" {
  description = "Proxmox ISO file ID"
  type        = string
}
```

Non-secret local environment values should be placed in `terraform.tfvars` or a non-committed local variable file.

Example:

```hcl
proxmox_host = "<proxmox-host-or-ip>"
proxmox_node = "<proxmox-node-name>"
vm_storage   = "<vm-storage-name>"
ubuntu_iso   = "<iso-storage>:iso/<ubuntu-server-iso>.iso"
```

Sensitive values should be provided through environment variables using Terraform's `TF_VAR_` convention.

Example:

```bash
export TF_VAR_proxmox_token_id="<token-id>"
export TF_VAR_proxmox_token_secret="<token-secret>"
```

## Terraform State Cleanup

Because the VM was disposable, the old Telmate resource was removed from state instead of importing the existing VM into BPG.

```bash
terraform state rm proxmox_vm_qemu.ubuntu_server_01
```

This removes Terraform tracking only. It does not delete the VM from Proxmox.

The old VM can then be removed manually from Proxmox before allowing BPG Terraform to rebuild it.

## Validation Commands

Run Terraform commands as the automation service account, not as a personal admin shell.

```bash
terraform init -upgrade
terraform validate
terraform providers
terraform plan
```

A clean provider result should show BPG and no Telmate provider requirement from state.

Expected provider shape:

```text
Providers required by configuration:
.
├── provider[registry.terraform.io/bpg/proxmox] ~> 0.104
└── provider[registry.terraform.io/hashicorp/local] ~> 2.5
```

## Permission Model

The Terraform project directory uses shared group access so both the service account and the admin user can work with project files.

Recommended pattern:

- Terraform code files: group-writable.
- Terraform state: owned and writable by the service account, not casually writable by the admin shell.
- Secret files: readable only by the service account.
- Project directory: setgid enabled so new files inherit the project group.
- ACLs or `umask 0002` used so new files are group-editable.

## GitHub Commit Guidance

Safe to commit:

```text
main.tf
variables.tf
ubuntu-server-01.tf
terraform.tfvars.example
.terraform.lock.hcl
.gitignore
README.md or documentation files
```

Do not commit:

```text
terraform.tfvars
terraform.tfstate
terraform.tfstate.*
.terraform/
backups/
*.env
secret files
vault password files
```

## Final Result

The project was successfully moved to BPG provider configuration. Terraform initialized and validated cleanly using BPG only, and the old Telmate state dependency was removed.

The next normal workflow is:

```bash
terraform plan
terraform apply
```

The apply should create the VM using BPG's `proxmox_virtual_environment_vm` resource.
