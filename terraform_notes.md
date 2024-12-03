ideme si teraz presunut state subor na remote obs bucket backend

uuidgen - for generation of e03f8044-f45f-45c5-abf3-6fabe2e6194e, unique ID
https://developer.hashicorp.com/terraform/language/backend

terraform.tf / backend.tf name of file as you want with .tf

terraform {
  backend "s3" {
    bucket                      = "obs-1000035578-terraform-states"
    key                         = "vyhonsky-e03f8044-f45f-45c5-abf3-6fabe2e6194e/terraform.tfstate"
    region                      = "eu-de"
    endpoint                    = "obs.eu-de.otc.t-systems.com"
    encrypt                     = false
    skip_credentials_validation = true
    skip_region_validation      = true
    skip_metadata_api_check     = true
  }
}

add to .bashrc and source it 

export AWS_SECRET_ACCESS_KEY="mbFrhqGJEKKnW2dt0DQuajkb2vgYiAyMYCe2uOWV"
export AWS_ACCESS_KEY_ID="KSAFW8YJXS0VG9ID5ZF1"

after that terraform init

now terraform state file will be stored remotely.
