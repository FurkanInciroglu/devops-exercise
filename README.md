Task3----

1-) Briefly, I would go with Prometheus/Grafana for simplicity. I would install node exporters on every single Ubuntu prod instances. Also I assume I have quite a lot prod instances so I would use Ansible/Chef as a configuration management tool to install node exporters.

2-) After that, I would install Prometheus on my center monitoring machine(which is not the same with my prod instances) to scrape my metrics which already exposed node exporter. Prometheus config file has to be created and need to specify scrape configs with node exporter IP and ports.

3-) Also Grafana can be install on the same monitoring machinbe with Prometheus. In Grafana, I’d configure a Prometheus data source pointing to the Prometheus server address.

4-) There are some useful ready Dashboards for multiple purpose on Grafana, I’d choose some of them and configure their promqls based on my needs to show data.

5-) At the last step, I need to create some alerts on Grafana to notify me in case of some incidents.

Task2------

1-)

For multiple environments in Terraform, I’d create Terraform
workspaces but also I see some downsides in my current experience to
use Terraform workspaces. Terraform workspaces allow us to isolate
environments but we need to create different tfvars files for our
different environments. Instead of that there are some additional
third party tools such as Terragrunt helps to achive that without
using Terraform workspaces.

Let’s assume we use Terraform workspaces. Of course as best practice
we would store our Terraform state files in remote state such as S3
bucket as an encrypted. Terraform doesn’t allow to define multiple s3
buckets for our different Terraform state files. It means we will use
the same bucket for prod and dev environment. It’s not a good
practice for security if someone get our terraform file would access
our prod terraform data.

Currently I solve this problem with using different backend tfvars.
Ex;

// prod-backend-config.tfvars

bucket = “my-prod-s3-bucket-for-terraform”
//
dev-backend-config.tfvars

bucket = “my-dev-s3-bucket-for-terraform”

And then inilizate our tf with $ terraform init -backend-config
prod-backend-config.tfvars

Stucturing with different env and different providers, Most people
prefer to use dev,qa,stage,prod different folders but I’d rather use
different tfvars to avoid duplication, otherwise I have to copy same
files in environment folders for all enviroments and I don’t want to
do that. So my layout approach roughly like below;

 terraform/
 ├── Application-Services/
 │   ├── Payment/
 |   |   ├── tfvars/
 |            ├── dev.tfvars
 |            ├── qa.tfvars
 |            ├── stage.tfvars
 |            ├── prod.tfvars
 │   │   ├── main.tf
 │   │   ├── variables.tf
 │   │   └── outputs.tf
 │   ├── Frontend/
 |   |   ├── tfvars/
 |            ├── dev.tfvars
 |            ├── qa.tfvars
 |            ├── stage.tfvars
 |            ├── prod.tfvars
 │   │   ├── main.tf
 │   │   ├── variables.tf
 │   │   └── outputs.tf
 │   ├── Backend/
 |   |   ├── tfvars/
 |            ├── dev.tfvars
 |            ├── qa.tfvars
 |            ├── stage.tfvars
 |            ├── prod.tfvars
 │   │   ├── main.tf
 │   │   ├── variables.tf
 │   │   └── outputs.tf
 ├── modules/
 │   ├── aws/
 │   │   ├── ec2/
 │   │   │   ├── main.tf
 │   │   │   ├── variables.tf
 │   │   │   └── outputs.tf
 │   │   ├── s3/
 │   │   │   ├── main.tf
 │   │   │   ├── variables.tf
 │   │   │   └── outputs.tf
 │   │   └── ...
 │   └── gcp/
 │       ├── vm/
 │       │   ├── main.tf
 │       │   ├── variables.tf
 │       │   └── outputs.tf
 │       ├── storage/
 │       │   ├── main.tf
 │       │   ├── variables.tf
 │       │   └── outputs.tf
 │       └── ...
 ├── providers.tf
 ├── variables.tf
 └── outputs.tf
 
2-) Assume I’ve already done AWS credentials and expose them on my machine which I run terraform commands or if I use Github/Gitlab I need to create secret and define my AWS credentials. It hepls me to set AWS cred whenever run runners/actions. Also creating ssh-key-pair to access our AWS instance need to be done before creating terraform script. I am not sure are we installing Ansible on the same place with my Terraform host or different machine(because I’ll use remote-exec or local-exec provisoner based on that) but I assume it’s different host so I use remote-exec for Ansible installation.

 		terraform {
 	  required_providers {
 	    aws = {
 	      source  = "hashicorp/aws"
 	      version = "=5.38"
 	    }
 	  }

 	  required_version = "= 1.7.3"
 	}

 	 provider "aws" {
 	   region  = "us-west-1"
 	}

 	 resource "aws_instance" "example_server" {
 	   ami           = "ami-06e45y7g3667g"
 	   instance_type = "m6g.large"
 	   key_name   = "my-key-pair"

 	   tags = {
 	     Name = "hello"
 	   }
 	}

 	 provisioner "remote-exec" {
 	    inline = ["sudo apt update", "sudo apt-get install -y ansible"]

 	    connection {
 	      host        = self.ipv4_address
 	      type        = "ssh"
 	      user        = "root"
 	      private_key = var.priv_key
 	    }
 	  }

 	  provisioner "local-exec" {
 	    command = "ansible-playbook -u root -i '${self.ipv4_address},' ansible-playbook.yml"
 	  }
}

Task5-----

In order to create Helm chart for this task, I’d create two folders “charts/” and “templates/”. In charts/ folder, I’ll create chart.yml for metadata and values.yml for the default configuration values for the Wordpress deployment. I can change values.yml based my custom configuration for example, if I want to increase the replicacount etc. I’ll define templates folder and files to generate deployment resources.

Chart.yml

apiVersion: v2

name: wordpress

version: 1.0.0 //chart version

description: WordPress application
Values.yml

	db:
  host: "mariadb"
  port: 3306
  username: "wordpress"
  password: "password"

wordpress:
  image: wordpress:latest
  replicaCount: 1
  resources:
    requests:
      memory: "1Gi"
      cpu: "1"

service:
  type: LoadBalancer
  port: 80

secret:
  name: wordpress-db-credentials
