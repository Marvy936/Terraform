
# Terraform Training Notes - Provider is openstack

## Table of Contents
1. [Introduction to Terraform Commands](#introduction-to-terraform-commands)
    - [Initialize Terraform](#initialize-terraform)
    - [Plan Terraform Execution](#plan-terraform-execution)
    - [Apply Terraform Plan](#apply-terraform-plan)
    - [Destroy Terraform Managed Infrastructure](#destroy-terraform-managed-infrastructure)
2. [Useful Links](#useful-links)
3. [Setting Up a Tenant](#setting-up-a-tenant)
4. [Exporting Proxy Variables](#exporting-proxy-variables)
5. [Creating a Keypair](#creating-a-keypair)
6. [Creating Security Groups](#creating-security-groups)
7. [Adding Security Group Rules](#adding-security-group-rules)
8. [Creating and Managing an Instance](#creating-and-managing-an-instance)

## Introduction to Terraform Commands

1. **Initialize Terraform**
   ```bash
   terraform init
   ```
   Initializes a new or existing Terraform configuration by preparing the working directory and downloading provider plugins.

2. **Plan Terraform Execution**
   ```bash
   terraform plan
   ```
   Creates an execution plan, showing what actions Terraform will take to achieve the desired state defined in your configuration.

3. **Apply Terraform Plan**
   ```bash
   terraform apply
   ```
   Executes the actions proposed in the plan to make your infrastructure match the desired state.

4. **Destroy Terraform Managed Infrastructure**
   ```bash
   terraform destroy
   ```
   Removes the resources created by Terraform.

## Useful Links

- [OpenTelekomCloud Provider Documentation](https://registry.terraform.io/providers/opentelekomcloud/opentelekomcloud/latest/docs)
- [Terraform Release Downloads](https://releases.hashicorp.com/)

## Setting Up a Tenant

1. Create a new directory:
   ```bash
   mkdir 00-tenant-
   ```
2. Create a `provider.tf` file with the following configuration:

   ```hcl
   terraform {
       required_version = ">= 0.14.0"
       required_providers {
           openstack = {
               source = "terraform-provider-openstack/openstack"
               version = "~> 1.48.0"
           }
       }
   }

   provider "openstack" {
       user_name        = "devops-terraform"
       password         = "2eir6shohHpeing3Mee]do"
       user_domain_name = "OTC-EU-DE-00000000001000035578"
       tenant_id        = "54026885c74446c2b833f4dc7cb77bd2"
       auth_url         = "https://iam.eu-de.otc.t-systems.com:443/v3"
       region           = "eu-de"
   }
   ```

3. Initialize Terraform:
   ```bash
   terraform init
   ```

## Exporting Proxy Variables (if needed)
```bash
export https_proxy=http://10.14.38.3:3128
export http_proxy=http://10.14.38.3:3128
```

## Creating a Keypair

1. Create a `keypair.tf` file with the following content:
   ```hcl
   resource "openstack_compute_keypair_v2" "vyhonsky-keypair" {
       name       = "vyhonsky-keypair"
       public_key = "{public_key}"
   }
   ```

2. Save the execution plan to a file:
   ```bash
   terraform plan -out=plan.tf
   ```

3. Apply the saved plan:
   ```bash
   terraform apply "plan.tf"
   ```

4. List Terraform-managed resources:
   ```bash
   terraform state list
   ```

## Creating Security Groups

1. Create a `security-group.tf` file:
   ```hcl
   resource "openstack_networking_secgroup_v2" "sg-vyhonsky" {
       name               = "sg-vyhonsky"
       description        = "ALL"
       delete_default_rules = true
   }
   ```

2. Save and apply the security group plan:
   ```bash
   terraform plan -out=scplan
   terraform apply "scplan"
   ```

## Adding Security Group Rules

1. Create a `sg-rule.tf` file:
   ```hcl
   resource "openstack_networking_secgroup_rule_v2" "allow_ssh_vyhonsky" {
       direction         = "ingress"
       ethertype         = "IPv4"
       protocol          = "tcp"
       port_range_min    = 22
       port_range_max    = 22
       remote_ip_prefix  = "10.0.0.0/8"
       security_group_id = openstack_networking_secgroup_v2.sg-vyhonsky.id
   }

   resource "openstack_networking_secgroup_rule_v2" "outbound_ssh_vyhonsky" {
       direction         = "egress"
       ethertype         = "IPv4"
       protocol          = "tcp"
       remote_ip_prefix  = "10.0.0.0/8"
       security_group_id = openstack_networking_secgroup_v2.sg-vyhonsky.id
   }
   ```

2. Apply the rules:
   ```bash
   terraform apply "scplan"
   ```

## Creating and Managing an Instance

1. Create a new directory:
   ```bash
   mkdir 01-single-instance
   cd 01-single-instance
   ```

2. Create an `instance.tf` file:
   ```hcl
   data "openstack_networking_secgroup_v2" "sg-vyhonsky" {
       name = "sg-vyhonsky"
   }

   data "openstack_compute_keypair_v2" "vyhonsky-keypair" {
       name = "vyhonsky-keypair"
   }

   resource "openstack_compute_instance_v2" "vyhonsky-vm" {
       name          = "vyhonsky-vm"
       image_name    = "Standard_Ubuntu_22.04_latest"
       flavor_name   = "s3.medium.2"
       key_pair      = data.openstack_compute_keypair_v2.vyhonsky-keypair.name
       user_data     = file("mount_VM.sh")

       network {
           port = openstack_networking_port_v2.primary_port.id
       }
   }
   ```

3. Create a volume and attach it to the instance:
   ```hcl
   resource "openstack_blockstorage_volume_v3" "data0-vyhonsky" {
       name               = "data0-vyhonsky"
       size               = 10
       enable_online_resize = true
       volume_type        = "SSD"
   }

   resource "openstack_compute_volume_attach_v2" "data0-vyhonsky" {
       instance_id = openstack_compute_instance_v2.vyhonsky-vm.id
       volume_id   = openstack_blockstorage_volume_v3.data0-vyhonsky.id
   }
   ```

4. Include a shell script `mount_VM.sh` for configuring the volume:
   ```bash
   #!/bin/bash

   mkfs -t ext4 /dev/vdb
   mkdir -p /mnt/data
   mount /dev/vdb /mnt/data
   echo /dev/vdb /mnt/data ext4 defaults,nofail 0 2 >> /etc/fstab
   ```
      After mount you have to resize disk.
   ```bash
   resize2fs /dev/vdb
   ```
   
