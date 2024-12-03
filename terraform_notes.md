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
  count       = length(local.ip_address)
  name                 = "data0-vyhonsky"
  size                 = 20
  enable_online_resize = true
  volume_type          = "SSD"
}

resource "openstack_networking_port_v2" "primary_port" {
  count       = length(local.ip_address)
  name       = format("port-%02d", count.index + 1)
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

resource "openstack_compute_instance_v2" "vyhonsky-vm" {
  count       = length(local.ip_address)
  name       = format("vyhonsky-vm-%02d", count.index + 1)
  image_name = "Standard_Ubuntu_22.04_latest"
  #image_name  =    "Enterprise_RedHat_9_latest"
  flavor_name = "s3.medium.2"
  key_pair    = data.openstack_compute_keypair_v2.vyhonsky-keypair.name
  #user_data   = file("mount_VM.sh")

  network {
    port = openstack_networking_port_v2.primary_port[count.index].id
  }
}

resource "openstack_compute_volume_attach_v2" "data0-vyhonsky" {
  count       = length(local.ip_address)
  instance_id = openstack_compute_instance_v2.vyhonsky-vm.id
  volume_id   = openstack_blockstorage_volume_v3.data0-vyhonsky.id
}

resource "null_resource" "provision" {
  count       = length(local.ip_address)
  triggers = {
    build_number = timestamp()
  }
  provisioner "local-exec" {
    command = "until ssh -o StrictHostKeyChecking=no -i ~/.ssh/mvyhonsk ubuntu@${openstack_compute_instance_v2.vyhonsky-vm[count.index].network[0].fixed_ip_v4}; do sleep 2;done && echo -e \"[server]\n${openstack_compute_instance_v2.vyhonsky-vm[count.index].network[0].fixed_ip_v4} ansible_ssh_private_key_file=~/.ssh/id_rsa ansible_ssh_common_args='-o StrictHostKeyChecking=no' ansible_ssh_user=ubuntu\" > inventory-test-${openstack_compute_instance_v2.vyhonsky-vm[count.index].name} &&  ansible-playbook -i inventory-test-${openstack_compute_instance_v2.vyhonsky-vm[count.index].name} install-httpd-ubuntu.yml"
  }
}





TEST



│ Error: Missing required argument
│
│   on instance.tf line 21, in resource "openstack_networking_port_v2" "primary_port":
│   21: resource "openstack_networking_port_v2" "primary_port" {
│
│ The argument "network_id" is required, but no definition was found.
╵
[mvyhonsk@bastion2 02-multiple-instances]$ vim instance.tf
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
  count       = length(local.ip_address)
  name                 = "data0-vyhonsky"
  size                 = 20
  enable_online_resize = true
  volume_type          = "SSD"
}

resource "openstack_networking_port_v2" "primary_port" {
  count       = length(local.ip_address)
  name       = format("port-%02d", count.index + 1)
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

resource "openstack_compute_instance_v2" "vyhonsky-vm" {
  count       = length(local.ip_address)
  name       = format("vyhonsky-vm-%02d", count.index + 1)
  image_name = "Standard_Ubuntu_22.04_latest"
  #image_name  =    "Enterprise_RedHat_9_latest"
  flavor_name = "s3.medium.2"
  key_pair    = data.openstack_compute_keypair_v2.vyhonsky-keypair.name
  #user_data   = file("mount_VM.sh")

  network {
    port = openstack_networking_port_v2.primary_port[count.index].id
  }
}

resource "openstack_compute_volume_attach_v2" "data0-vyhonsky" {
  count       = length(local.ip_address)
  instance_id = openstack_compute_instance_v2.vyhonsky-vm.id
  volume_id   = openstack_blockstorage_volume_v3.data0-vyhonsky.id
}

resource "null_resource" "provision" {
  count       = length(local.ip_address)
  triggers = {
    build_number = timestamp()
  }
  provisioner "local-exec" {
    command = "until ssh -o StrictHostKeyChecking=no -i ~/.ssh/mvyhonsk ubuntu@${openstack_compute_instance_v2.vyhonsky-vm[count.index].network[0].fixed_ip_v4}; do sleep 2;done && echo -e \"[server]\n${openstack_compute_instance_v2.vyhonsky-vm[count.index].network[0].fixed_ip_v4} ansible_ssh_private_key_file=~/.ssh/id_rsa ansible_ssh_common_args='-o StrictHostKeyChecking=no' ansible_ssh_user=ubuntu\" > inventory-test-${openstack_compute_instance_v2.vyhonsky-vm[count.index].name} &&  ansible-playbook -i inventory-test-${openstack_compute_instance_v2.vyhonsky-vm[count.index].name} install-httpd-ubuntu.yml"
  }
}
