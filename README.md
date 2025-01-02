# Modularize Terraform Project

This project demonstrates how to modularize Terraform configurations by dividing resources into reusable modules. Modularizing Terraform projects improves readability, maintainability, and reusability of your infrastructure code.

## Technologies Used
- **Terraform**
- **AWS**
- **Docker**
- **Linux**
- **Git**

---

## Project Description
This project modularizes a Terraform setup by creating two modules:
1. **Subnet Module**: Responsible for subnet-related resources.
2. **Webserver Module**: Responsible for EC2 instance-related resources.

---

## Steps to Modularize Terraform Resources

### Step 1: Move Providers, Variables, and Outputs to Separate Files
1. Create the following files for organization:
   - `providers.tf`: Define Terraform providers.
   - `variables.tf`: Define all input variables.
   - `outputs.tf`: Define output variables.

**Example Code:**

#### `providers.tf`
```hcl
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "4.67.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

#### `variables.tf`
```hcl
variable "env_prefix" {}
variable "avail_zone" {}
variable "vpc_cidr_block" {}
variable "subnet_cidr_block" {}
variable "my_ip" {}
variable "instance_type" {}
variable "public_key_location" {}
```

#### `outputs.tf`
```hcl
output "aws_ami_id" {
  value = data.aws_ami.latest-amazon-linux-image.id
}

output "ec2_public_ip" {
  value = aws_instance.myapp-server.public_ip
}
```

Move all remaining resources and data definitions into `main.tf`.

---

### Step 2: Create Folder Structure for Modules
1. Create a `modules` directory with subdirectories for each module.
2. Set up `main.tf`, `variables.tf`, and `outputs.tf` for each module.

**Commands:**
```bash
mkdir -p modules/subnet modules/webserver

# Subnet module
touch modules/subnet/{main.tf,variables.tf,outputs.tf}

# Webserver module
touch modules/webserver/{main.tf,variables.tf,outputs.tf}
```

---

### Step 3: Move resources related to Subnet
1. Move subnet-related resources from `main.tf` to `modules/subnet/main.tf`.
2. Replace references to parent resources with variables in `modules/subnet/variables.tf`.

**Example:**

#### `modules/subnet/main.tf`
```hcl
resource "aws_subnet" "myapp-subnet-1" {
  vpc_id = var.vpc_id
  cidr_block = var.subnet_cidr_block
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

#### `modules/subnet/variables.tf`
```hcl
variable "env_prefix" {}
variable "avail_zone" {}
variable "subnet_cidr_block" {}
variable "vpc_id" {}
variable "default_route_table_id" {}
```

---

### Step 4: Reference the Subnet Module
1. Add a `module` block to reference the subnet module in `main.tf`.

**Example:**

#### `main.tf`
```hcl
module "myapp-subnet" {
  source = "./modules/subnet"

  env_prefix = var.env_prefix
  avail_zone = var.avail_zone
  subnet_cidr_block = var.subnet_cidr_block
  vpc_id = aws_vpc.myapp-vpc.id
  default_route_table_id = aws_vpc.myapp-vpc.default_route_table_id
}
```

---

### Step 5: Specify module outputs to fix any broken references
1. Add outputs for resources in the `subnet` module.
2. Reference these outputs in the root module.

**Example:**

#### `modules/subnet/outputs.tf`
```hcl
output "subnet" {
  value = aws_subnet.myapp-subnet-1
}
```

#### `main.tf`
```hcl
resource "aws_instance" "myapp-server" {
  subnet_id = module.myapp-subnet.subnet.id
}
```

---

### Step 6: Create the Webserver Module
1. Extract EC2-related resources into the `webserver` module.
2. Define required variables and outputs.

**Example:**

#### `modules/webserver/main.tf`
```hcl
resource "aws_instance" "myapp-server" {
  ami = data.aws_ami.latest-amazon-linux-image.id
  instance_type = var.instance_type
  subnet_id = var.subnet_id
  vpc_security_group_ids = [aws_default_security_group.default-sg.id]
  key_name = aws_key_pair.ssh-key.key_name
  tags = {
    Name = "${var.env_prefix}-server"
  }
}
```

#### `modules/webserver/variables.tf`
```hcl
variable "env_prefix" {}
variable "subnet_id" {}
variable "instance_type" {}
variable "public_key_location" {}
```

---

### Step 7: Reference the Webserver Module
Add a `module` block to the root `main.tf` to reference the `webserver` module.

**Example:**

#### `main.tf`
```hcl
module "myapp-server" {
  source = "./modules/webserver"
  env_prefix = var.env_prefix
  subnet_id = module.myapp-subnet.subnet.id
  instance_type = var.instance_type
  public_key_location = var.public_key_location
}
```

---

### Step 8: Specify Output references
Update `outputs.tf` to reference outputs from the webserver module.

#### `modules/webserver/outputs.tf`
```hcl
output "instance" {
  value = aws_instance.myapp-server
}
```

#### `outputs.tf`
```hcl
output "ec2_public_ip" {
  value = module.myapp-server.instance.public_ip
}
```

---

### Step 9: Apply the Configuration Changes
1. Reinitialize Terraform to incorporate module changes:
   ```bash
   terraform init
   ```
2. Validate and apply the changes:
   ```bash
   terraform plan
   terraform apply --auto-approve
   ```

---

### Step 10: SSH into the EC2 Instance
1. Use the public IP to access the EC2 instance.
2. Verify Docker and nginx are running:
   ```bash
   docker ps
   
