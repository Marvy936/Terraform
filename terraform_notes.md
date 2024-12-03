https://registry.terraform.io/providers/hashicorp/null/latest/docs/resources/resource

resource "null_resource" "provision" {
  triggers = {
    build_number = timestamp()
  }

https://developer.hashicorp.com/terraform/language/resources/provisioners/remote-exec


provisioner and remote exec

resource "null_resource" "provision" {
  triggers = {
    build_number = timestamp()
  }

  connection {
    user        = "ubuntu"
    private_key = file("/home/mvyhonsk/.ssh/mvyhonsk")
    host        = openstack_compute_instance_v2.vyhonsky-vm.network.0.fixed_ip_v4
  }

  provisioner "remote-exec" {
    inline = [
      "echo Hello World!",
      "date",
      "ls -la",
    ]
  }
}


local exec

https://developer.hashicorp.com/terraform/language/resources/provisioners/local-exec


install-httpd-ubuntu.yml
----------------------
- hosts: server
  remote_user: ubuntu
  become: true
  tasks:
  - name: Configure proxy
    blockinfile:
      path: /etc/environment
      block: |
        http_proxy="http://10.14.38.3:3128"
        https_proxy="http://10.14.38.3:3128"
    ignore_errors: true
  - name: Install Apache2 Webserver
    apt: name=apache2 state=latest
  - name: Enable Apache on system reboot
    service: name=apache2 enabled=yes
    notify: restart apache
  - name: Update apt cache
    apt:
      update_cache: yes
  - name: Install links package
    apt:
      name: links
      state: present
  - name: Verify links installation
    command: links -version
    register: links_version
    failed_when: false
    changed_when: false
  - name: Show links version installed
    debug:
      msg: "Links version installed: {{ links_version.stdout }}"
  handlers:
  - name: restart apache
    service: name=apache2 state=restarted



resource "null_resource" "provision" {
  triggers = {
    build_number = timestamp()
  }
  provisioner "local-exec" {
    command = "until ssh -o StrictHostKeyChecking=no -i ~/.ssh/mvyhonsk ubuntu@${local.ip_address}; do sleep 2;done && echo -e \"[server]\n${openstack_compute_instance_v2.vyhonsky-vm.network[0].fixed_ip_v4} ansible_ssh_private_key_file=~/.ssh/mvyhonsk ansible_ssh_common_args='-o StrictHostKeyChecking=no' ansible_ssh_user=ubuntu\" > inventory-test &&  ansible-playbook -i inventory-test install-httpd-ubuntu.yml"
  }
}


useful functions to manipulate data: https://developer.hashicorp.com/terraform/language/functions daj priklad na element co pouzijeme a lenght

meta argument count
https://developer.hashicorp.com/terraform/language/meta-arguments/count

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

resource "openstack_networking_secgroup_rule_v2" "healthcheck_http_apache" {
  count             = length(local.default_to_apache)
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = element(local.default_to_apache, count.index)
  port_range_max    = element(local.default_to_apache, count.index)
  remote_ip_prefix  = "100.125.0.0/16"
  security_group_id = openstack_networking_secgroup_v2.sg-vyhonsky.id
}





