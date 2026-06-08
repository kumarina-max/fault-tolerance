## Домашнее задание к занятию «Отказоустойчивость в облаке»

## Задание  1

## Развертывание инфраструктуры в Yandex Cloud с помощью Terraform

##  Описание

Terraform-конфигурация создаёт:
- сеть и подсеть;
- две виртуальные машины с Ubuntu 22.04 и автоматически установленным Nginx;
- целевую группу для балансировщика;
- сетевой балансировщик нагрузки, слушающий порт 80 и проверяющий состояние ВМ через HTTP healthcheck.

## 📁 Файлы конфигурации

### `main.tf` – основной манифест

```hcl
terraform {
  required_version = ">= 1.3"
  required_providers {
    yandex = {
      source  = "yandex-cloud/yandex"
      version = ">= 0.98.0"
    }
  }
}

provider "yandex" {
  token     = var.yc_token
  cloud_id  = var.yc_cloud_id
  folder_id = var.yc_folder_id
  zone      = var.yc_zone
}

data "yandex_compute_image" "ubuntu" {
  family = "ubuntu-2204-lts"
}

resource "yandex_vpc_network" "default" {
  name = "lb-network"
}

resource "yandex_vpc_subnet" "default" {
  name           = "lb-subnet"
  zone           = var.yc_zone
  network_id     = yandex_vpc_network.default.id
  v4_cidr_blocks = ["10.128.0.0/24"]
}

resource "yandex_compute_instance" "web" {
  count = 2

  name        = "web-${count.index + 1}"
  platform_id = "standard-v2"
  zone        = var.yc_zone

  resources {
    cores  = 2
    memory = 2
  }

  boot_disk {
    initialize_params {
      image_id = data.yandex_compute_image.ubuntu.image_id
      size     = 20
    }
  }

  network_interface {
    subnet_id = yandex_vpc_subnet.default.id
    nat       = true
  }

  metadata = {
    user-data = <<-EOF
      #cloud-config
      packages:
        - nginx
      runcmd:
        - systemctl enable nginx
        - systemctl start nginx
    EOF
  }
}

resource "yandex_lb_target_group" "web_group" {
  name      = "web-target-group"
  region_id = "ru-central1"

  dynamic "target" {
    for_each = yandex_compute_instance.web
    content {
      subnet_id = yandex_vpc_subnet.default.id
      address   = target.value.network_interface[0].ip_address
    }
  }
}

resource "yandex_lb_network_load_balancer" "web_lb" {
  name = "web-balancer"

  listener {
    name        = "http-listener"
    port        = 80
    target_port = 80
    external_address_spec {
      ip_version = "ipv4"
    }
  }

  attached_target_group {
    target_group_id = yandex_lb_target_group.web_group.id

    healthcheck {
      name = "http-healthcheck"
      http_options {
        port = 80
        path = "/"
      }
    }
  }
}

output "web_external_ips" {
  value = yandex_compute_instance.web[*].network_interface[0].nat_ip_address
}

output "load_balancer_ip" {
  value = one(flatten([
    for listener in yandex_lb_network_load_balancer.web_lb.listener : [
      for spec in listener.external_address_spec : spec.address
    ]
  ]))
}
```
### variables.tf – переменные

```hcl
variable "yc_token" {
  description = "OAuth‑токен Яндекс.Облака"
  type        = string
  sensitive   = true
}

variable "yc_cloud_id" {
  description = "Идентификатор облака"
  type        = string
}

variable "yc_folder_id" {
  description = "Идентификатор каталога"
  type        = string
}

variable "yc_zone" {
  description = "Зона доступности"
  type        = string
  default     = "ru-central1-a"
}

variable "image_id" {
  description = "ID образа Ubuntu 22.04 LTS"
  type        = string
  default     = "fd8ne5h1q2n76djsj68i"  # Ubuntu 22.04 LTS в ru-central1-a
}
```

