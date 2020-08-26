# terraform-google-panos-bootstrap
This Terraform Module creates a PAN-OS bootstrap package in a Google Cloud Storage bucket to be used for bootstrapping Palo Alto Networks VM-Series virtual firewall instances.  A bootstrap package must include an `init-cfg.txt` file that provides the basic configuration details to configure the VM-Series instance and register it with its Panorama management console.  This file will be generated by this module using the variables provided.  

The bootstrap package may optionally include a PAN-OS software image, application and threat signature updates, VM-Series plug-ins, and/or license files.

## Directory and file structure
The root directory of the Terraform plan calling this module should include a `files` directory containing a subdirectory structure similar to the one below.

```
files
├── config
├── content
├── license
├── plugins
└── software
```

## Example

```terraform
#
# main.tf
#

provider "google" {
  credentials = file("account.json")
  project     = var.bootstrap_project
  region      = var.bootstrap_region
}

module "panos-bootstrap" {
  source  = "PaloAltoNetworks/panos-bootstrap/google"
  version = "1.0.0"

  bootstrap_project     = var.bootstrap_project
  bootstrap_region      = var.bootstrap_region

  hostname         = "my-firewall"
  panorama-server  = "panorama1.example.org"
  panorama-server2 = "panorama2.example.org"
  tplname          = "My Firewall Template"
  dgname           = "My Firewalls"
  vm-auth-key      = "supersecretauthkey"
}
```

## Instructions

1. Define a `main.tf` file that calls the module and provides any required and optional variables.
2. Define a `variables.tf` file that declares the variables that will be utilized.
3. (OPTIONAL) Define an `output.tf` file to capture and display the module return values.
4. Create the directories `files/config`, `files/software`, `files/content`, `files/license`, and `files/plugins`.
5. (OPTIONAL) Add software images, content updates, plugins, and license files to their respective subdirectories.
6. (OPTIONAL) Define a `terraform.tfvars` file containing the required variables and associated values.
7. Initialize the providers and modules with the `terraform init` command.
8. Validate the plan using the `terraform plan` command.
9. Apply the plan using the `terraform apply` command. 

## Utilization

The module output will provide values for the `bootstrap_name` and `bootstrap_url`, and `share_name`.  The `bootstrap_name` value can then be used in a `google_compute_instance` resource to instantiate a VM-Series instance.  It is used in the `metadata{vmseries-bootstrap-gce-storagebucket}` parameter.

```terraform
resource "google_compute_instance" "firewall" {
	name						= var.fw_name
	zone						= var.fw_zone
	machine_type				= var.fw_machine_type
	min_cpu_platform			= var.fw_machine_cpu
	can_ip_forward				= true
	allow_stopping_for_update	= true
	count						= 1

	boot_disk {
		initialize_params {
			image				= var.fw_image
		}
	}

	metadata = {
		vmseries-bootstrap-gce-storagebucket = module.panos-bootstrap.bootstrap_name
		serial-port-enable		= true
		block-project-ssh-keys	= true
		ssh-keys				= var.fw_ssh_key
	}

	service_account {
		scopes = ["cloud-platform"]
	}

	network_interface {
		subnetwork				= var.fw_mgmt_subnet
		network_ip				= var.fw_mgmt_ip
		access_config {
			// Needed to get a public IP address
		}
	}

	network_interface {
		subnetwork				= var.fw_untrust_subnet
		network_ip				= var.fw_untrust_ip
		access_config {
			// Needed to get a public IP address
		}
	}

	network_interface {
		subnetwork				= var.fw_web_subnet
		network_ip				= var.fw_web_ip
	}

	network_interface {
		subnetwork				= var.fw_db_subnet
		network_ip				= var.fw_db_ip
	}
}
```


## References
* [VM-Series Firewall Bootstrap Workflow](https://docs.paloaltonetworks.com/vm-series/10-0/vm-series-deployment/bootstrap-the-vm-series-firewall/vm-series-firewall-bootstrap-workflow.html#id59fe5979-c29d-42aa-8e72-14a2c12855f6)
* [Bootstrap the VM-Series Firewall on Google Cloud Platform](https://docs.paloaltonetworks.com/vm-series/10-0/vm-series-deployment/bootstrap-the-vm-series-firewall/bootstrap-the-vm-series-firewall-on-google.html#id17CRC0V0TR0)
* [Prepare the Bootstrap Package](https://docs.paloaltonetworks.com/vm-series/10-0/vm-series-deployment/bootstrap-the-vm-series-firewall/prepare-the-bootstrap-package.html#id5575318c-1de8-497a-960a-1d7417feefa6)