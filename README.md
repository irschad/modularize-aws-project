# Modularize AWS Infrastructure with Terraform

## GitHub Repository
[Modularize AWS Project Repository](https://github.com/irschad/modularize-aws-project)

This project demonstrates how to modularize AWS infrastructure provisioning with Terraform. By dividing resources into reusable modules, the configuration is more organized, maintainable, and scalable.

---

## Project Overview
### Objectives
1. **Modularization:** Separate Terraform resources into reusable modules.
2. **Infrastructure Automation:** Provision AWS infrastructure for subnets and EC2 instances.
3. **Dockerized Application Deployment:** Launch a web server using Docker running on an EC2 instance.

### Key Features
- **Subnet Module:** Automates the creation of a subnet, internet gateway, and route table.
- **Web Server Module:** Automates the setup of an EC2 instance, security group, and associated configurations.
- **Custom Scripts:** Includes a bash script for Docker installation and container deployment.

### Technologies Used
- **Terraform**
- **AWS**
- **Docker**
- **Linux**
- **Git**

---

## Folder Structure

### Root Directory
- **main.tf:** Root configuration for calling modules and setting up the VPC.
- **variables.tf:** Centralized variable declarations.
- **outputs.tf:** Root-level outputs for the infrastructure.
- **providers.tf:** AWS provider configuration.
- **entry-script.sh:** Shell script for setting up the web server with Docker.

### Modules
1. **Subnet Module:**
   - **Location:** `modules/subnet`
   - Files: `main.tf`, `variables.tf`, `outputs.tf`
   - Provisions a subnet, internet gateway, and default route table.

2. **Web Server Module:**
   - **Location:** `modules/webserver`
   - Files: `main.tf`, `variables.tf`, `outputs.tf`
   - Provisions an EC2 instance, security group, and key pair.

---

## Detailed Breakdown

### Subnet Module
**File:** `modules/subnet/main.tf`
```hcl
resource "aws_subnet" "myapp-subnet-1" {
    vpc_id            = var.vpc_id
    cidr_block        = var.subnet_cidr_block
    availability_zone = var.avail_zone
    tags = {
        Name = "${var.env_prefix}-subnet-1"
    }
}

resource "aws_internet_gateway" "myapp-igw" {
    vpc_id = var.vpc_id
    tags = {
        Name = "${var.env_prefix}-igw"
    }
}

resource "aws_default_route_table" "main-rtb" {
    default_route_table_id = var.default_route_table_id

    route {
        cidr_block = "0.0.0.0/0"
        gateway_id = aws_internet_gateway.myapp-igw.id
    }
    tags = {
        Name = "${var.env_prefix}-main-rtb"
    }
}
```

**File:** `modules/subnet/variables.tf`
```hcl
variable env_prefix {}
variable avail_zone {}
variable subnet_cidr_block {}
variable vpc_id {}
variable default_route_table_id {}
```

**File:** `modules/subnet/outputs.tf`
```hcl
output "subnet" {
  value = aws_subnet.myapp-subnet-1
}
```

### Web Server Module
**File:** `modules/webserver/main.tf`
```hcl
resource "aws_default_security_group" "default-sg" {
    vpc_id = var.vpc_id

    ingress {
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = [var.my_ip]
    }

    ingress {
        from_port   = 8080
        to_port     = 8080
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }

    egress {
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
        cidr_blocks = ["0.0.0.0/0"]
    }

    tags = {
        Name = "${var.env_prefix}-default-sg"
    }
}

resource "aws_instance" "myapp-server" {
    ami                         = data.aws_ami.latest-amazon-linux-image.id
    instance_type               = var.instance_type
    subnet_id                   = var.subnet_id
    vpc_security_group_ids      = [aws_default_security_group.default-sg.id]
    availability_zone           = var.avail_zone
    associate_public_ip_address = true
    key_name                    = aws_key_pair.ssh-key.key_name

    user_data                  = file("entry-script.sh")
    user_data_replace_on_change = true

    tags = {
        Name = "${var.env_prefix}-server"
    }
}
```

**File:** `modules/webserver/variables.tf`
```hcl
variable env_prefix {}
variable avail_zone {}
variable my_ip {}
variable instance_type {}
variable public_key_location {}
variable vpc_id {}
variable image_name {}
variable subnet_id {}
```

**File:** `modules/webserver/outputs.tf`
```hcl
output "instance" {
  value = aws_instance.myapp-server
}
```

---

## Steps to Run the Project

1. **Clone the Repository**
   ```bash
   git clone https://github.com/irschad/modularize-aws-project.git
   cd modularize-aws-project
   ```

2. **Initialize Terraform**
   ```bash
   terraform init
   ```

3. **Plan the Configuration**
   ```bash
   terraform plan
   ```

4. **Apply the Configuration**
   ```bash
   terraform apply
   ```

5. **Access the Deployed Application**
   Retrieve the public IP of the EC2 instance from the outputs and access it via the browser on port 8080.

---

## Outputs
- **Subnet ID:** Outputs the ID of the created subnet.
- **EC2 Public IP:** Outputs the public IP of the deployed EC2 instance.

---



