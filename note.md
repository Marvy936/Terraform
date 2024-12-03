# Terraform Training Notes

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
9. [How to Create Resources and Reference Them Using `data`](#how-to-create-resources-and-reference-them-using-data)
10. [Variables in Terraform](#variables-in-terraform)
    - [Input Variables](#input-variables)
        - [Declaring Variables](#declaring-variables)
        - [Using Input Variables](#using-input-variables)
    - [Output Variables](#output-variables)
    - [Local Values](#local-values)
        - [Declaring Local Values](#declaring-local-values)
        - [Using Local Values](#using-local-values)
        - [Ephemeral Local Values](#ephemeral-local-values)
        - [Best Practices](#best-practices-for-variables-and-local-values)
11. [Backend](#backend)
12. [Null Resource](#null-resource)
     - [Triggers Example](#triggers-example)
13. [Provisioners](#provisioners)
     - [Remote Exec Provisioner](#remote-exec-provisioner)
     - [Local Exec Provisioner](#local-exec-provisioner)
14. [Useful Terraform Functions](#useful-terraform-functions)
     - [Element Function](#element-function)
     - [Length Function](#length-function)
15. [Meta-Argument: Count](#meta-argument-count)
16. [Terraform Code Explanation: Multiple Instances with `count`](#terraform-code-explanation-multiple-instances-with-count)
    - [Key Concept: `count`](#key-concept-count)
    - [Resource-by-Resource Breakdown](#resource-by-resource-breakdown)
         - [1. Security Groups (`data` blocks)](#1-security-groups-data-blocks)
         - [2. Keypair (`data` block)](#2-keypair-data-block)
         - [3. Block Storage Volume](#3-block-storage-volume)
         - [4. Networking Port](#4-networking-port)
         - [5. Compute Instances](#5-compute-instances)
         - [6. Volume Attachment](#6-volume-attachment)
         - [7. Provisioning (`null_resource`)](#7-provisioning-null_resource)
    - [Dynamic Provisioning with `count`](#dynamic-provisioning-with-count)
        - [1. IP List (`local.ip_address`)](#1-ip-list-localip_address)
        - [2. Dynamic Resource Naming](#2-dynamic-resource-naming)
        - [3. Resource Dependencies](#3-resource-dependencies)

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

- [OpenStack Provider Documentation](https://registry.terraform.io/providers/terraform-provider-openstack/openstack/latest/docs)
- [OpenTelekomCloud Provider Documentation](https://registry.terraform.io/providers/opentelekomcloud/opentelekomcloud/latest/docs)
- [Terraform Release Downloads](https://releases.hashicorp.com/)

## Setting Up a Tenant

1. Create a new directory:
   ```bash
   mkdir 00-tenant-base
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
export https_proxy=http://00.00.00.0:0000
export http_proxy=http://00.00.00.0:0000
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
       user_data     = file("path/to/script")

       network {
           port = openstack_networking_port_v2.primary_port.id
       }
   }
   ```
   
---
   
## How to Create Resources and Reference Them Using `data`

1. Creating Security Group and Key Pair:

To later fetch a security group with a data block, you first create it as a resource.

```hcl
resource "openstack_networking_secgroup_v2" "sg-vyhonsky" {
    name               = "sg-vyhonsky"
    description        = "Security group for Vyhonsky"
    delete_default_rules = true
}

resource "openstack_compute_keypair_v2" "vyhonsky-keypair" {
    name       = "vyhonsky-keypair"
    public_key = file("~/.ssh/id_rsa.pub")
}   
```

2. Referencing and Using Them in Other Resources:

Once resources like a security group or key pair are created, you can use data blocks to fetch their attributes dynamically. This is useful when the exact resource may be used across multiple configurations or modules.

```hcl
data "openstack_networking_secgroup_v2" "sg-vyhonsky" {
    name = openstack_networking_secgroup_v2.sg-vyhonsky.name
}

data "openstack_compute_keypair_v2" "vyhonsky-keypair" {
    name = openstack_compute_keypair_v2.vyhonsky-keypair.name
}

resource "openstack_compute_instance_v2" "vyhonsky-vm" {
    name       = "vyhonsky-vm"
    image_name = "Standard_Ubuntu_22.04_latest"
    flavor_name = "s3.medium.2"
    key_pair   = data.openstack_compute_keypair_v2.vyhonsky-keypair.name

    network {
        port = openstack_networking_port_v2.primary_port.id
    }
}
```

The `user_data` field in the openstack_compute_instance_v2 resource refers to a mechanism used to initialize a virtual machine (VM) during its first boot. 
It allows you to specify a script or set of commands that will be executed as part of the VM's startup process.

---
   
4. Create a volume and attach it to the instance:
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

5. Include a shell script `mount_VM.sh` for configuring the volume:
   ```bash
   #!/bin/bash

   mkfs -t ext4 /dev/vdb
   mkdir -p /mnt/data
   mount /dev/vdb /mnt/data
   echo /dev/vdb /mnt/data ext4 defaults,nofail 0 2 >> /etc/fstab
   ```
After resizing volume, you have to resize mounted disk.
   ```bash
   resize2fs /dev/vdb
   ```
   
## Variables in Terraform

`https://upcloud.com/docs/guides/terraform-variables/`

Variables in Terraform enable dynamic and reusable configurations. They are declared in `variables.tf` files or directly within your Terraform manifests.

### Input Variables

Input variables are values that you pass into Terraform configurations.

#### Declaring Variables

Variables must first be defined in a block or `variables.tf` file:
```hcl
variable "private_key_path" {
  type    = string
  default = "/home/user/.ssh/terraform_rsa"
}

variable "public_key" {
  type    = string
  default = "ssh-rsa terraform_public_key"
}

variable "zones" {
  type = map
  default = {
    "amsterdam" = "nl-ams1"
    "london"    = "uk-lon1"
    "frankfurt" = "de-fra1"
    "helsinki1" = "fi-hel1"
    "helsinki2" = "fi-hel2"
    "chicago"   = "us-chi1"
    "sanjose"   = "us-sjo1"
    "singapore" = "sg-sin1"
  }
}

variable "plans" {
  type = map
  default = {
    "5USD"  = "1xCPU-1GB"
    "10USD" = "1xCPU-2GB"
    "20USD" = "2xCPU-4GB"
  }
}

variable "storage_sizes" {
  type = map
  default = {
    "1xCPU-1GB" = "25"
    "1xCPU-2GB" = "50"
    "2xCPU-4GB" = "80"
  }
}

variable "templates" {
  type = map
  default = {
    "ubuntu18" = "01000000-0000-4000-8000-000030080200"
    "centos7"  = "01000000-0000-4000-8000-000050010300"
    "debian9"  = "01000000-0000-4000-8000-000020040100"
  }
}

variable "set_password" {
  type    = bool
  default = false
}

variable "users" {
  type    = list
  default = ["root", "user1", "user2"]
}

variable "plan" {
  type    = string
  default = "10USD"
}

variable "template" {
  type    = string
  default = "ubuntu18"
}
```

#### Using Input Variables

Input variables can be referenced in configurations:
```hcl
var.image_name
```

You can pass values through:
- **Files (`terraform.tfvars.json`)**:
  ```json
  {
    "private_key_path": "VALUE",
    "private_key_path": "VALUE"
  }
  ```
  Files named `terraform.tfvars`, `terraform.tfvars.json`, or ending with `.auto.tfvars / .auto.tfvars.json` are automatically loaded.

- **Command-line Flags**:
  ```bash
  terraform apply -var set_password="true"
  ```

- **Environment Variables**: Use the `TF_VAR_` prefix to define variables as environment variables:
  ```bash
  export TF_VAR_PASSWORD="password"
  ```
  Ensure the variable is declared in the configuration:
  ```hcl
  variable "PASSWORD" {
    default = ""
  }
  ```

### Output Variables

Output variables allow you to extract information from Terraform's state:
```hcl
output "public_ip" {
  value = upcloud_server.server_name.network_interface[0].ip_address
}
```

To access output variables, use:
```bash
terraform output {output_var_name}
```

### Local Values

Local values assign names to expressions for reuse within a module.

#### Declaring Local Values `locals.tf`

```hcl
locals {
  service_name = "forum"
  owner        = "Community Team"
  common_tags  = {
    Service = local.service_name
    Owner   = local.owner
  }
}
```

#### Using Local Values

`https://developer.hashicorp.com/terraform/language/values/locals`

Local values can be referenced as `local.<name>`:
```hcl
resource "aws_instance" "example" {
  tags = local.common_tags
}
```

#### Ephemeral Local Values (Terraform v1.10+)

Ephemeral local values depend on ephemeral inputs:
```hcl
variable "service_token" {
  type      = string
  ephemeral = true
}

locals {
  session_token = "${var.service_name}:${var.service_token}"
}
```

## Best Practices for Variables and Local Values

- Use input variables for configuration flexibility.
- Use local values to reduce repetition but avoid overuse for readability.
- Keep sensitive variables in environment variables or securely managed files.

## Backend

`https://developer.hashicorp.com/terraform/language/backend`

For generating uniqe ID use:
```bash
uuidgen
```
Create a new file, such as `backend.tf` or `terraform.tf` or whatever you want ending with `.tf` and add the following content to configure the backend: (this is configuration for OTC)

```hcl
terraform {
  backend "s3" {
    bucket                  = "obs-1000035578-terraform-states"
    key                     = "vyhonsky-e03f8044-f45f-45c5-abf3-6fabe2e6194e/terraform.tfstate"
    region                  = "eu-de"
    endpoint                = "obs.eu-de.otc.t-systems.com"
    encrypt                 = false
    skip_credentials_validation = true
    skip_region_validation      = true
    skip_metadata_api_check     = true
  }
}
```

`key = "vyhonsky-e03f8044-f45f-45c5-abf3-6fabe2e6194e/terraform.tfstate"` -> this is the path where you state will be stored on OTC bucket.

Add your credentials to `.bashrc` and `source ~/.bashrc` for easy reuse:

```bash
export AWS_SECRET_ACCESS_KEY="your_secret_access_key"
export AWS_ACCESS_KEY_ID="your_access_key_id"
```
Now just run `terraform init` command.

## Null Resource

The `null_resource` is a Terraform resource that does not manage any actual infrastructure. It is often used for tasks like triggering provisioners or managing dependencies.

### Triggers Example

```hcl
resource "null_resource" "provision" {
  triggers = {
    build_number = timestamp()
  }
}
```

- **`triggers`**: Defines conditions that trigger the execution of provisioners.
  - **`build_number`**: A custom key to store a value that determines when the resource runs.
  - **`timestamp()`**: A built-in Terraform function that returns the current time in UTC.

---

## Provisioners

Provisioners allow you to execute scripts or commands on a local or remote machine during Terraform execution. There are two main types: `remote-exec` and `local-exec`.

### Remote Exec Provisioner

The `remote-exec` provisioner executes scripts on a remote instance.

#### Example

```hcl
resource "null_resource" "provision" {
  triggers = {
    build_number = timestamp()
  }

  connection {
    user        = "ubuntu"
    private_key = file("/home/mvyhonsk/.ssh/mvyhonsk")
    host        = openstack_compute_instance_v2.vyhonsky-vm.network[0].fixed_ip_v4
  }

  provisioner "remote-exec" {
    inline = [
      "echo Hello World!",
      "date",
      "ls -la",
    ]
  }
}
```

#### Explanation of Connection Block

1. **`user`**: Specifies the username for SSH access.
   - Example: `user = "ubuntu"`
   - Use the default username for the instance's operating system.

2. **`private_key`**: Specifies the private SSH key for authentication.
   - Example: `private_key = file("/home/mvyhonsk/.ssh/mvyhonsk")`
   - Provide the path to your private key.

3. **`host`**: The IP address or hostname of the remote instance.
   - Example: `host = openstack_compute_instance_v2.vyhonsky-vm.network[0].fixed_ip_v4`
   - Use the public or private IP of the instance.

4. **`inline`**: A list of commands to run on the remote server.
   - Example:
     - `"echo Hello World!"`: Prints "Hello World!".
     - `"date"`: Displays the current date and time.
     - `"ls -la"`: Lists files and directories in long format.

---

### Local Exec Provisioner

The `local-exec` provisioner runs commands on the local machine executing Terraform.

#### Example

```hcl
resource "null_resource" "provision" {
  triggers = {
    build_number = timestamp()
  }

  provisioner "local-exec" {
    command = <<EOT
      until ssh -o StrictHostKeyChecking=no -i ~/.ssh/mvyhonsk ubuntu@${local.ip_address}; do sleep 2; done &&
      echo -e "[server]\n${openstack_compute_instance_v2.vyhonsky-vm.network[0].fixed_ip_v4} ansible_ssh_private_key_file=~/.ssh/mvyhonsk ansible_ssh_common_args='-o StrictHostKeyChecking=no' ansible_ssh_user=ubuntu" > inventory-test &&
      ansible-playbook -i inventory-test install-httpd-ubuntu.yml
    EOT
  }
}
```

#### Explanation of the Command

1. **SSH Connection Loop**
   ```bash
   until ssh -o StrictHostKeyChecking=no -i ~/.ssh/mvyhonsk ubuntu@${local.ip_address}; do sleep 2; done
   ```
   - Attempts SSH connection every 2 seconds until successful.

2. **Create Ansible Inventory**
   ```bash
   echo -e "[server]\n${openstack_compute_instance_v2.vyhonsky-vm.network[0].fixed_ip_v4} ansible_ssh_private_key_file=~/.ssh/mvyhonsk ansible_ssh_common_args='-o StrictHostKeyChecking=no' ansible_ssh_user=ubuntu" > inventory-test
   ```
   - Dynamically generates an inventory file for Ansible.

3. **Run Ansible Playbook**
   ```bash
   ansible-playbook -i inventory-test install-httpd-ubuntu.yml
   ```
   - Executes the Ansible playbook to configure the remote server.

---

## Useful Terraform Functions

Terraform functions allow data manipulation and processing.

### Element Function

The `element` function retrieves an item from a list by index.

#### Example

```hcl
locals {
  ports = [80, 443, 8080]
}

resource "openstack_networking_secgroup_rule_v2" "default_ports" {
  count             = length(local.ports)
  port_range_min    = element(local.ports, count.index)
  port_range_max    = element(local.ports, count.index)
}
```

### Length Function

The `length` function returns the number of items in a list or map.

#### Example

```hcl
locals {
  ports = [80, 443, 8080]
}

resource "null_resource" "example" {
  count = length(local.ports)
}
```

---

## Meta-Argument: Count

The `count` meta-argument allows creating multiple instances of a resource.

### Example

```hcl
locals {
  default_to_apache = [80, 443, 8080]
}

resource "openstack_networking_secgroup_rule_v2" "default_to_apache" {
  count             = length(local.default_to_apache)
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = element(local.default_to_apache, count.index)
  port_range_max    = element(local.default_to_apache, count.index)
  remote_ip_prefix  = "10.0.0.0/8"
  security_group_id = openstack_networking_secgroup_v2.sg-vyhonsky.id
}
```

#### Explanation

1. **`count`**: Creates one resource per port in `local.default_to_apache`.
2. **`element`**: Dynamically assigns each port to the resource.

---

# References

- [Null Resource Documentation](https://registry.terraform.io/providers/hashicorp/null/latest/docs/resources/resource)
- [Remote Exec Provisioner](https://developer.hashicorp.com/terraform/language/resources/provisioners/remote-exec)
- [Local Exec Provisioner](https://developer.hashicorp.com/terraform/language/resources/provisioners/local-exec)
- [Terraform Functions](https://developer.hashicorp.com/terraform/language/functions)
- [Meta-Argument Count](https://developer.hashicorp.com/terraform/language/meta-arguments/count)

---

# Terraform Code Explanation: Multiple Instances with `count`

This Terraform configuration dynamically provisions multiple OpenStack resources based on the number of IP addresses defined in the `local.ip_address` variable. Here's a detailed breakdown of the code:

---

## Key Concept: `count`
- The `count` meta-argument in Terraform creates multiple instances of a resource dynamically.
- The number of instances is determined by `count`, which in this case is set to `length(local.ip_address)`—one instance per IP in the `local.ip_address` list.

---

## Resource-by-Resource Breakdown

### 1. Security Groups (`data` blocks)
```hcl
data "openstack_networking_secgroup_v2" "sg-AgileAcademyTelIT-default" {
  name = "sg-AgileAcademyTelIT-default"
}

data "openstack_networking_secgroup_v2" "sg-vyhonsky" {
  name = "sg-vyhonsky"
}
```
- These `data` blocks retrieve details about existing security groups in OpenStack.
- They are used in the `security_group_ids` parameter of the networking port to associate security groups with the port.

---

### 2. Keypair (`data` block)
```hcl
data "openstack_compute_keypair_v2" "vyhonsky-keypair" {
  name = "vyhonsky-keypair"
}
```
- Retrieves the details of an existing key pair (`vyhonsky-keypair`) in OpenStack.
- The key pair is used to configure SSH access for the compute instances.

---

### 3. Block Storage Volume
```hcl
resource "openstack_blockstorage_volume_v3" "data0-vyhonsky" {
  count       = length(local.ip_address)
  name        = "data0-vyhonsky"
  size        = 20
  enable_online_resize = true
  volume_type = "SSD"
}
```
- Creates one block storage volume per IP address.
- `count.index` can be used to make each name unique, e.g., `data0-vyhonsky-%02d`.

---

### 4. Networking Port
```hcl
resource "openstack_networking_port_v2" "primary_port" {
  count       = length(local.ip_address)
  network_id  = local.network_id
  name        = format("port-%02d", count.index + 1)
  security_group_ids = [
    data.openstack_networking_secgroup_v2.sg-vyhonsky.id,
    data.openstack_networking_secgroup_v2.sg-AgileAcademyTelIT-default.id,
  ]
  admin_state_up = "true"
  fixed_ip {
    subnet_id  = local.subnet_id
    ip_address = local.ip_address[count.index]
  }
}
```
- Creates a networking port for each IP address.
- Assigns a specific IP from the `local.ip_address` list to the port.

---

### 5. Compute Instances
```hcl
resource "openstack_compute_instance_v2" "vyhonsky-vm" {
  count       = length(local.ip_address)
  name        = format("vyhonsky-vm-%02d", count.index + 1)
  image_name  = "Standard_Ubuntu_22.04_latest"
  flavor_name = "s3.medium.2"
  key_pair    = data.openstack_compute_keypair_v2.vyhonsky-keypair.name

  network {
    port = openstack_networking_port_v2.primary_port[count.index].id
  }
}
```
- Provisions a compute instance for each IP address.
- Dynamically names each instance using `count.index`.

---

### 6. Volume Attachment
```hcl
resource "openstack_compute_volume_attach_v2" "data0-vyhonsky" {
  count       = length(local.ip_address)
  instance_id = openstack_compute_instance_v2.vyhonsky-vm[count.index].id
  volume_id   = openstack_blockstorage_volume_v3.data0-vyhonsky[count.index].id
}
```
- Attaches a block storage volume to each compute instance.
- The correct volume is matched with the correct instance using `count.index`.

---

### 7. Provisioning (`null_resource`)
```hcl
resource "null_resource" "provision" {
  count       = length(local.ip_address)
  triggers = {
    build_number = timestamp()
  }
  provisioner "local-exec" {
    command = "until ssh -o StrictHostKeyChecking=no -i ~/.ssh/mvyhonsk ubuntu@${openstack_compute_instance_v2.vyhonsky-vm[count.index].network[0].fixed_ip_v4}; do sleep 2;done && echo -e "[server]\n${openstack_compute_instance_v2.vyhonsky-vm[count.index].network[0].fixed_ip_v4} ansible_ssh_private_key_file=~/.ssh/mvyhonsk ansible_ssh_common_args='-o StrictHostKeyChecking=no' ansible_ssh_user=ubuntu" > inventory-test-${openstack_compute_instance_v2.vyhonsky-vm[count.index].name} &&  ansible-playbook -i inventory-test-${openstack_compute_instance_v2.vyhonsky-vm[count.index].name} install-httpd-ubuntu.yml"
  }
}
```
- Provisions each instance using a local command.
- SSH is used to configure the instance, and an Ansible playbook is executed for additional setup.

---

## Dynamic Provisioning with `count`
### 1. IP List (`local.ip_address`)
- Controls how many instances and resources are created. Each IP corresponds to a full set of resources (port, volume, compute instance, etc.).
- Example:
  ```hcl
  locals {
    ip_address = ["192.168.1.101", "192.168.1.102"]
  }
  ```

### 2. Dynamic Resource Naming
- Resources are uniquely named using `count.index`, ensuring no conflicts.
- Example:
  ```hcl
  format("vyhonsky-vm-%02d", count.index + 1)
  ```

### 3. Resource Dependencies
- Resources are linked via `count.index`, ensuring each resource (e.g., port, volume) is associated with the correct instance.

This design allows you to scale up or down by modifying the `local.ip_address` list and running `terraform apply`. Each IP corresponds to one full stack of resources.
