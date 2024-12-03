variables

terraform.tfvars.json

"image_name": "VAULE"

output variables

output "public_ip" {
  value = upcloud_server.server_name.network_interface[0].ip_address
}

terraform output {output_var_name}

input variables - my vkladame hodnotuT


musim mat v bloku variable zadefinovnae premenne bud v jednotlivych manuifestovch alebo v samostatnom variables.tf

variable "private_key_path" {
  type = string
  default = "/home/user/.ssh/terraform_rsa"
}

variable "public_key" {
  type = string
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
  type = bool
  default = false
}

variable "users" {
  type = list
  default = ["root", "user1", "user2"]
}

variable "plan" {
  type = string
  default = "10USD"
}

variable "template" {
  type = string
  default = "ubuntu18"
}

potom mozem do json file urcit hodnoty 

terraform.tfvars.json - files named exactly terraform.tfvars or terraform.tfvars.json.
Any files with names ending in .auto.tfvars or .auto.tfvars.json. are automaticly loaded

"image_name": "VAULE"
"private_key_pat": "new value"

input variables - my vkladame hodnotuT

potom ich mozem pouzivat var.image_name v kode.


mozem hodnotu urcit aj terraform apply -var set_password="true" but again it have to be set first in block variable or variables.tf

```
Using environmental variables
You can also set sensitive variables in your environment variables with the TF_VAR_ prefix avoiding the need to save them in a file. For example, set your password in your local environmental variables.

export TF_VAR_PASSWORD="password"

You’ll also need to declare the password variable in your variables.tf file.

variable PASSWORD { default = "" }

The password variable is then usable in the Terraform resources.

  provisioner "remote-exec" {
    inline = [
      "useradd ${var.users[0]}",
      "echo '${var.users[0]}:${var.PASSWORD}' | chpasswd"
    ]
  }

When deployed, the remote execution provisioner will create a new user according to the users variable with the PASSWORD as set in the environmental variable.


```

variables we can use between different terraform folders

Local Values
Hands-on: Try the Simplify Terraform Configuration with Locals tutorial.

A local value assigns a name to an expression, so you can use the name multiple times within a module instead of repeating the expression.

If you're familiar with traditional programming languages, it can be useful to compare Terraform modules to function definitions:

Input variables are like function arguments.
Output values are like function return values.
Local values are like a function's temporary local variables.
Note: For brevity, local values are often referred to as just "locals" when the meaning is clear from context.

Declaring a Local Value
A set of related local values can be declared together in a single locals block:

locals {
  service_name = "forum"
  owner        = "Community Team"
}

The expressions in local values are not limited to literal constants; they can also reference other values in the module in order to transform or combine them, including variables, resource attributes, or other local values:

locals {
  # Ids for multiple sets of EC2 instances, merged together
  instance_ids = concat(aws_instance.blue.*.id, aws_instance.green.*.id)
}

locals {
  # Common tags to be assigned to all resources
  common_tags = {
    Service = local.service_name
    Owner   = local.owner
  }
}

Ephemeral values
Note: Ephemeral local values are available in Terraform v1.10 and later.

Local values implicitly become ephemeral if you reference an ephemeral value when you assign that local a value. For example, you can create a local that references an ephemeral service_token.

variable "service_name" {
  type    = string
  default = "forum"
}

variable "environment" {
  type    = string
  default = "dev"
}

variable "service_token" {
  type      = string
  ephemeral = true
}

locals {
  service_tag   = "${var.service_name}-${var.environment}"
  session_token = "${var.service_name}:${var.service_token}"
}

The local.session_token value is implicitly ephemeral because it relies on an ephemeral variable.

Using Local Values
Once a local value is declared, you can reference it in expressions as local.<NAME>.

Note: Local values are created by a locals block (plural), but you reference them as attributes on an object named local (singular). Make sure to leave off the "s" when referencing a local value!

resource "aws_instance" "example" {
  # ...

  tags = local.common_tags
}

A local value can only be accessed in expressions within the module where it was declared.

When To Use Local Values
Local values can be helpful to avoid repeating the same values or expressions multiple times in a configuration, but if overused they can also make a configuration hard to read by future maintainers by hiding the actual values used.

Use local values only in moderation, in situations where a single value or result is used in many places and that value is likely to be changed in future. The ability to easily change the value in a central place is the key advantage of local values.
