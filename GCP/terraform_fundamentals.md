# Terraform Fundaentals Lab

(simple instance configuration - instance.tf )

```
resource "google_compute_instance" "terraform" {
  project      = "<PROJECT_ID>"
  name         = "terraform"
  machine_type = "n1-standard-1"
  zone         = "us-west1-c"
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }
  network_interface {
    network = "default"
    access_config {
    }
  }
}
```

terraform basic commands

```
terraform init
```

```
terraform plan
```

```
terraform apply
```

```
terraform show
```

touch main.tf

```
terraform {
  required_providers {
    google = {
      source = "hashicorp/google"
    }
  }
}
provider "google" {
  version = "3.5.0"
  project = "<PROJECT_ID>"
  region  = "us-central1"
  zone    = "us-central1-c"
}
resource "google_compute_network" "vpc_network" {
  name = "terraform-network"
}
resource "google_compute_instance" "vm_instance" {
  name         = "terraform-instance"
  machine_type = "f1-micro"
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }
  network_interface {
    network = google_compute_network.vpc_network.name
    access_config {
    }
  }
}
```

(delete whole infra)

```
terraform destroy
```

(static ip configuration)

```
resource "google_compute_address" "vm_static_ip" {
  name = "terraform-static-ip"
}
```

(terraform plan with -out option )

```
terraform plan -out static_ip
```

(terraform apply with plan from -out option)

```
terraform apply "static_ip"
```

(depends on added)

```
resource "google_storage_bucket" "example_bucket2" {
  name     = "qwiklabs-gcp-00-82aa0fef2b82"
  depends_on = [
    google_storage_bucket.example_bucket
  ]
}
```

(provisioner added)

```
resource "google_compute_instance" "vm_instance" {
  name         = "terraform-instance"
  machine_type = "f1-micro"
  tags         = ["web", "dev"]
  provisioner "local-exec" {
    command = "echo ${google_compute_instance.vm_instance.name}:  ${google_compute_instance.vm_instance.network_interface[0].access_config[0].nat_ip} >> ip_address.txt"
    }
  boot_disk {
    initialize_params {
      image = "cos-cloud/cos-stable"
    }
  }

 network_interface {
    network = google_compute_network.vpc_network.self_link
    access_config {
      nat_ip = google_compute_address.vm_static_ip.address
    }
  }
}
```
