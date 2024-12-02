tereraform init

terraform plan

terraform apply

terraform destroy

https://registry.terraform.io/providers/opentelekomcloud/opentelekomcloud/latest/docs

https://releases.hashicorp.com/

mkdir 00-tenant-

-----------------
create provider.tf
--------------------
terraform {
required_version = ">= 0.14.0"
  required_providers {
    openstack = {
      source  = "terraform-provider-openstack/openstack"
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
  #version = "< 1.21.0"
}

terraform init


export https_proxy=http://10.14.38.3:3128
export http_proxy=http://10.14.38.3:3128


---------------------
keypair.tf
--------------------
resource "openstack_compute_keypair_v2" "vyhonsky-keypair" {
  name       = "vyhonsky-keypair"
  public_key = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHBC81O6rZHjpDa8HpOb+IGzNDoW6KcnrvsloCs2Iudk martin.vyhonsky@telekom.com"
}


terraform plan
-out={saved_file_name} -> save plan to file

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # openstack_compute_keypair_v2.vyhonsky-keypair will be created
  + resource "openstack_compute_keypair_v2" "vyhonsky-keypair" {
      + fingerprint = (known after apply)
      + id          = (known after apply)
      + name        = "vyhonsky-keypair"
      + private_key = (known after apply)
      + public_key  = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHBC81O6rZHjpDa8HpOb+IGzNDoW6KcnrvsloCs2Iudk martin.vyhonsky@telekom.com"
      + region      = (known after apply)
      + user_id     = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

When used option -out

Saved the plan to: plan.tf

To perform exactly these actions, run the following command to apply:
    terraform apply "plan.tf"


  terraform apply "plan.tf"


terraform state list -> shows resources from state file

now we are going to create security group 

security-group.tf

resource "openstack_networking_secgroup_v2" "sg-vyhonsky" {
  name                 = "sg-vyhonsky"
  description          = "ALL"
  delete_default_rules = true
}

terraform plan out=scplan

terraform apply "scplan"

now we are going to create rules for security group 

sg-rule.tf

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

terraform show



mkdir 01-single-instance
cd 01-single-instance

pomocoou data viem nacitavat uz vytvorene resource ako keypair ktory sme si vytvoirili, securtiy group nakolko to nie je v jednom priecinku musime pouzit data.                          

instance.tf


data "openstack_networking_secgroup_v2" "sg-AgileAcademyTelIT-default" {
  name = "sg-AgileAcademyTelIT-default"
}

data "openstack_networking_secgroup_v2" "sg-vyhonsky" {
  name = "sg-vyhonsky"
}

data "openstack_compute_keypair_v2" "vyhonsky-keypair" {
  name = "vyhonsky-keypair"
}

resource "openstack_blockstorage_volume_v3" "data0-vyhonsky" {
  name                 = "data0-vyhonsky"
  size                 = 10
  enable_online_resize = true
  volume_type          = "SSD"
}

resource "openstack_networking_port_v2" "primary_port" {
  network_id = "9e322103-7a52-4e15-b667-2ea8e2ca41ce"
  security_group_ids = [
    data.openstack_networking_secgroup_v2.sg-vyhonsky.id,
    data.openstack_networking_secgroup_v2.sg-AgileAcademyTelIT-default.id,
  ]
  admin_state_up = "true"
  fixed_ip {
    subnet_id  = "45463fd3-7491-4dcc-9040-b5ce6602da09"
    ip_address = "10.14.253.32"
  }
}

resource "openstack_compute_instance_v2" "vyhonsky-vm" {
  name       = "vyhonsky-vm"
  image_name = "Standard_Ubuntu_22.04_latest"
  #image_name  =    "Enterprise_RedHat_9_latest"
  flavor_name = "s3.medium.2"
  key_pair    = data.openstack_compute_keypair_v2.vyhonsky-keypair.name
  user_data   = file("mount_VM.sh")

  network {
    port = openstack_networking_port_v2.primary_port.id
  }
}

resource "openstack_compute_volume_attach_v2" "data0-vyhonsky" {
  instance_id = openstack_compute_instance_v2.vyhonsky-vm.id
  volume_id   = openstack_blockstorage_volume_v3.data0-vyhonsky.id
}

mount_VM.sh

#!/bin/bash
#
mkfs -t ext4 /dev/vdb
mkdir -p /mnt/data
mount /dev/vdb /mnt/data
echo /dev/vdb /mnt/data ext4 defaults,nofail 0 2 >> /etc/fstab




