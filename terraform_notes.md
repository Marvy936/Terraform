Terraform Variables and Local Values
Variables in Terraform

Variables in Terraform enable dynamic and reusable configurations. They are declared in variables.tf files or directly within your Terraform manifests.
Input Variables

Input variables are values that you pass into Terraform configurations.
Declaring Variables

Variables must first be defined in a block or variables.tf file:

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

Using Input Variables

Input variables can be referenced in configurations:

var.image_name

You can pass values through:

    Files (terraform.tfvars.json):

{
  "image_name": "VALUE",
  "private_key_path": "new value"
}

Files named terraform.tfvars, terraform.tfvars.json, or ending with .auto.tfvars/.auto.tfvars.json are automatically loaded.

Command-line Flags:

terraform apply -var set_password="true"

Environment Variables: Use the TF_VAR_ prefix to define variables as environment variables:

export TF_VAR_PASSWORD="password"

Ensure the variable is declared in the configuration:

    variable "PASSWORD" {
      default = ""
    }

Output Variables

Output variables allow you to extract information from Terraform's state:

output "public_ip" {
  value = upcloud_server.server_name.network_interface[0].ip_address
}

To access output variables, use:

terraform output {output_var_name}

Local Values

Local values assign names to expressions for reuse within a module.
Declaring Local Values

locals {
  service_name = "forum"
  owner        = "Community Team"
  common_tags  = {
    Service = local.service_name
    Owner   = local.owner
  }
}

Using Local Values

Local values can be referenced as local.<name>:

resource "aws_instance" "example" {
  tags = local.common_tags
}

Ephemeral Local Values (Terraform v1.10+)

Ephemeral local values depend on ephemeral inputs:

variable "service_token" {
  type      = string
  ephemeral = true
}

locals {
  session_token = "${var.service_name}:${var.service_token}"
}

Best Practices for Variables and Local Values

    Use input variables for configuration flexibility.
    Use local values to reduce repetition but avoid overuse for readability.
    Keep sensitive variables in environment variables or securely managed files.
