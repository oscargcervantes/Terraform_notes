# Terransible

---

## Spin up an Ubuntu 16.04 server

Create the server and get the details under LinuxAcademy

---

## Prerequisites

* Python 2
* apt-get update
* apt-install python-pip

### Install Terraform, Ansible and AWS cli

    curl -O https://releases.hashicorp.com/terraform/0.11.2/terraform_0.11.2_linux_amd64.zip
    mkdir bin/terraform
    unzip terraform_0.11.2_linux_amd64.zip -d /bin/terraform
    export PATH=$PATH:/bin/terraform
    terraform --version
    
    pip install aws-cli --upgrade
    aws --version
    
    apt-get install software-properties-common
    apt-add-repository pp:ansible/ansible
    apt-get update
    apt-get install ansible #Or can use pip to install it
    ansible --version
    
    ssh-agent bash
    ssh-add ~/.ssh/id_rsa
    ssh-add -l
    
    vi /etc/ansible/ansible.cfg #uncomment host_key_checking
    
    mkdir terransible
    cd terransible
    
---

## IAM and DNS setup

### IAM

* Go to AWS console, IAM service and create a user named terransible with only programmatic access
* Assign Admin access permission to this user
* Download the credentials and save them

### Route53 - Optional

    aws route53 create-reusable-delegation-set --caller-reference 1224 --profile superhero #Save the output
    
### Add credential to use on aws-cli

    aws configure --profile superhero #Provide the credentials
    aws ec2 describe instances --profile superhero
    
---

## Credentials and variables

### main.tf, terraform.tfvars, variables.tf    

#### main.tf
    
    provider "aws" {
        region  = "${vars.aws_region}"
        profile = "${vars.aws_profile}"
    }

#### variables.tf

    variable "aws_region" {}
    variable "aws_profile" {}

#### terraform.tfvars

    aws_profile = "superhero"
    aws_region  = "us-east-2"

---

## Terraform Init and IAM

    terraform init
    terraform plan
    
### main.tf

    provider "aws" {
        region  = "${vars.aws_region}"
        profile = "${vars.aws_profile}"
    }
    
    #------IAM-------
    
    #S3_access
    
    resource "aws_iam_instance_profile" "s3_access_profile" {
        name = "s3_access"
        role = "${aws_iam_role.s3_access_role.name}"
    }
    
    resource "aws_iam_role_policy" "s3_access_policy" {
        name = "s3_access_policy"
        role = "${aws_iam_role.s3_access_role.id}"
        
        policy = <<EOF
        {
          "Version": "2012-10-17",
          "Statement": [
          {
              "Effect": "Allow",
              "Action": "s3:*",
              "Resource": "*"
          }
         ]
        }
        EOF
        
    }
    
    resource "aws_iam_role" "s3_access_role" {
        name = "s3_access_role"
        
        assume_role_policy = <<EOF
        {
          "Version": "2012-10-17",
          "Statement": [
          {
            "Action": "sts:AssumeRole",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Effect": "Allow",
            "Sid": ""
          }
         ]
        }
        EOF
    }
    
### Plan

    terraform plan

---

## Creating the VPC part1

### Add to main.tf

    #------VPC------
    
    resource "aws_vpc" "wp_vpc" {
        cidr_block = "${var.vpc_cidr}"
        enable_dns_hostnames = true
        enable_dns_support   = true
        
        tags {
          Name = "wp_vpc"
        }
        
    }
    
    #---internet gateway---
    
    resource "aws_internet_gateway" "wp_internet_gateway" {
        vpc_id = "${aws_vpc.wp_vpc.id}"
        
        tags {
          Name = "wp_igw"
        }   
    }
    
    ##---Route tables---
    
    resource "aws_route_table" "wp_public_rt" {
        vpc_id = "${aws_vpc.wp_vpc.id}"
        
        route {
          cidr_block = "0.0.0.0/0"               
          gateway_id = "${aws_internet_gateway.wp_internet_gateway.id}"
        }
        
        tags {
          Name = "wp_public"
        }
    }
    
    resource "aws_default_route_table" "wp_private_rt" {
        default_route_table_id = "${aws_vpc.wp_vpc.default_route_table_id}"
        
        tags {
          Name = "wp_private"
        }
    }
              
### Add to variables.tf

    data "aws_availability_zones" "available" {}
    variable "vpc_cidr" {} 
          
### Add to terraform.tfvars

    vpc_cidr = "10.0.0.0/16"            
              
---

## Creating the VPC part2 - Subnets

### Add to main.tf

    #------Subnets------
    
    resource "aws_subnet" "wp_public1_subnet" {
        vpc_id = "${aws_vpc.wp_vpc.id}"
        cidr_block = "${var.cidrs["public1"]}" #map type variable
        map_public_ip_on_launch = true
        availability_zone = "${data.aws.availability_zones.available.names[0]}"
        
        tag {
          Name = "wp_public1"
        }
    }
    
    resource "aws_subnet" "wp_public2_subnet" {
        vpc_id = "${aws_vpc.wp_vpc.id}"
        cidr_block = "${var.cidrs["public2"]}" #map type variable
        map_public_ip_on_launch = true
        availability_zone = "${data.aws.availability_zones.available.names[1]}"
        
        tag {
          Name = "wp_public2"
        }
    }
    
    resource "aws_subnet" "wp_private1_subnet" {
        vpc_id = "${aws_vpc.wp_vpc.id}"
        cidr_block = "${var.cidrs["private1"]}" #map type variable
        map_public_ip_on_launch = false #Because it is a private subnet
        availability_zone = "${data.aws.availability_zones.available.names[0]}"
        
        tag {
          Name = "wp_private1"
        }
    }
    
    resource "aws_subnet" "wp_private2_subnet" {
        vpc_id = "${aws_vpc.wp_vpc.id}"
        cidr_block = "${var.cidrs["private2"]}" #map type variable
        map_public_ip_on_launch = false #Because it is a private subnet
        availability_zone = "${data.aws.availability_zones.available.names[1]}"
        
        tag {
          Name = "wp_private2"
        }
    }
    
    resource "aws_subnet" "wp_rds1_subnet" {
        vpc_id = "${aws_vpc.wp_vpc.id}"
        cidr_block = "${var.cidrs["rds1"]}" #map type variable
        map_public_ip_on_launch = false #Because it is a private subnet
        availability_zone = "${data.aws.availability_zones.available.names[0]}"
        
        tag {
          Name = "wp_rds1"
        }
    }
    
    resource "aws_subnet" "wp_rds2_subnet" {
        vpc_id = "${aws_vpc.wp_vpc.id}"
        cidr_block = "${var.cidrs["rds2"]}" #map type variable
        map_public_ip_on_launch = false #Because it is a private subnet
        availability_zone = "${data.aws.availability_zones.available.names[1]}"
        
        tag {
          Name = "wp_rds2"
        }
    }
    
    resource "aws_subnet" "wp_rds3_subnet" {
        vpc_id = "${aws_vpc.wp_vpc.id}"
        cidr_block = "${var.cidrs["rds3"]}" #map type variable
        map_public_ip_on_launch = false #Because it is a private subnet
        availability_zone = "${data.aws.availability_zones.available.names[2]}"
        
        tag {
          Name = "wp_rds3"
        }
    }
             
### Add to terraform.tfvars

    cidrs = {
      public1   = "10.0.1.0/24"
      public2   = "10.0.2.0/24"
      private1  = "10.0.3.0/24"
      private2  = "10.0.4.0/24"
      rds1      = "10.0.5.0/24"
      rds2      = "10.0.6.0/24"
      rds3      = "10.0.7.0/24"
    }

### Add tovariables.tf

     variable "cidrs" {
       type = "map"
     }
   
---

## Creating the VPC part3 - Subnet Group and Route Associations

### Add to main.tf

    #------RDS Subnet group------    
    
    resource "aws_db_subnet_group" "wp_rds_subnetgroup" {
        name = "wp_rds_subnetgroup"
        
        subnet_ids = ["${aws_subnet_wp_rds1_subnet.id}",
                      "${aws_subnet_wp_rds2_subnet.id}",
                      "${aws_subnet_wp_rds3_subnet.id}"
                    ]
        
        tags = {
          Name = "wp_rds_sng"
        }
    }
    
    #------Subnet associations------
    
    resource "aws_route_table_association" "wp_public1_assoc" {
        subnet_id = "${aws_subnet.wp_public1_subnet.id}"
        route_table_id = "${aws_route_table.wp_public_rt.id}"
    }
    
    resource "aws_route_table_association" "wp_public2_assoc" {
        subnet_id = "${aws_subnet.wp_public2_subnet.id}"
        route_table_id = "${aws_route_table.wp_public_rt.id}"
    }
    
    resource "aws_route_table_association" "wp_private1_assoc" {
        subnet_id = "${aws_subnet.wp_private1_subnet.id}"
        route_table_id = "${aws_default_route_table.wp_private_rt.id}"
    }
     
    resource "aws_route_table_association" "wp_private2_assoc" {
        subnet_id = "${aws_subnet.wp_private2_subnet.id}"
        route_table_id = "${aws_default_route_table.wp_private_rt.id}"
    }
    
### Format 

    terraform fmt

---

## Creating the security groups

### Add to main.tf

    #------Security Groups------
    
    resource "aws_security_group" "wp_dev_sg" {
        name = "wp_dev_sg"
        description = "Used for access to the dev instance"
        vpc_id = "${aws_vpc.wp_vpc.id}"
        
        #SSH
        
        ingress {
          from_port = 22
          to_port = 22
          protocol = "tcp"
          cidr_blocks = ["${var.localip}"]
        }
        
        #HTTP
        
        ingress {
          from_port = 80
          to_port = 80
          protocol = "tcp"
          cidr_blocks = ["0.0.0.0/0"] #["${var.localip}"]
        }
        
        egress {
          from_port = 0 #Means everything
          to_port = 0
          protocol = "-1" #All protocols
          cidr_blocks = ["0.0.0.0/0"]
        }
    }
    
    #------Public security groups------
    
    resource "aws_security_group" "wp_public_sg" {
        name = "wp_public_sg"
        description = "Used for the elastic load balancer for public access"
        vpc_id = "${aws_vpc.wp_vpc.id}"
        
        #HTTP
        
        ingress {
          from_port = 80
          to_port = 80
          protocol = "tcp"
          cidr_blocks = ["0.0.0.0/0"]
        }
        
        egress {
          from_port = 0 #Means everything
          to_port = 0
          protocol = "-1" #All protocols
          cidr_blocks = ["0.0.0.0/0"]
        }
    }
    
    #------Private security groups------
    
    resource "aws_security_group" "wp_private_sg" {
        name = "wp_private_sg"
        description = "Used for private instances"
        vpc_id = "${aws_vpc.wp_vpc.id}"
        
        #HTTP
        
        ingress {
          from_port = 80
          to_port = 80
          protocol = "tcp"
          cidr_blocks = ["${var.vpc_cidr}"]
        }
        
        egress {
          from_port = 0 #Means everything
          to_port = 0
          protocol = "-1" #All protocols
          cidr_blocks = ["0.0.0.0/0"]
        }
    }
    
    #------RDS security group------
    
    resource "aws_security_group" "wp_rds_sg" {
        name = "wp_rds_sg"
        description = "Used for RDS instances"
        vpc_id = "${aws_vpc.wp_vpc.id}"
        
        #SQL access from public/private security groups
        
        ingress {
          from_port = 3306
          to_port = 3306
          protocol = "tcp"
          
          security_groups = ["${aws_security_group.wp_dev_sg.id}",
            "${aws_security_group.wp_public_sg.id}",
            "${aws_security_group.wp_private_sg.id}"
          ]
        }
    }       
    
### Add to terraform.tfvars

    localip = "54.210.164.210/32" #IP of my Ubuntu VM 

### Add to variables.tf       
        
     variable "localip" {} 

---

## S3 Bucket creation

### Add to main.tf

    #------VPC endpoint for S3
    
    resource "aws_vpc_endpoint" "wp_private-s3_endpoint" {
        vpc_id = "${aws_vpc.wp_vpc.id}"
        service_name = "com.amazonaws.${var.aws_region}.s3"
        
        route_table_ids = ["${aw_s_vpc.wp_vpc.main_route_table_id}",
          "${aws_route_table.wp_public_rt.id}"
        ]
        
        policy = <<POLICY
        {
          "Statement": [
            {
                "Action": "*",
                "Effect": "Allow",
                "Resource": "*",
                "Principal": "*"
            }
          ]
        }
        POLICY
    }
    
    #------S3 code bucket------
    
    resource "random_id" "wp_code_bucket" {
        byte_length = 2 #6 digits                
    }
    
    resource "aws_s3_bucket" "code" {
        bucket = "${var.domain_name}-${random_id.wp_code_bucket.dec}"
        acl = "private"
        force_destroy = true #S3 bucket delete if other infra is destroyed
        
        tags {
          Name = "code bucket"
        }
    }

### Add to terraform.tfvars

    domain_name = "dreamerscloud" #My domain

### Add to variables.tf

    variable "domain_name" {}

### Install random provider to terraform

    terraform init
    terraform plan 
    terrafomr fmt

---

## Creating the RDS instance
    
### Add to main.tf

    #------RDS intance------
    
    resource "aws_db_instance" "wp_db" {
        allocated_storage = 10 #GB
        engine = "mysql"
        engine_version = "5.6.27"
        instance_class = "${var.db_instance_class}" #Size of the server for the DB
        name = "${var.dbname}"
        username = "${var.dbuser}"
        password = "${var.db.password}"
        db_subnet_group_name = "${aws_db_subnet_group.wp_rds_subnetgroup.name}"
        vpc_security_group_ids = ["${aws_security_group.wp_rds_sg.id}"]
        skip_final_snapshot = true #To avoid errors
    }
    
### Add to terraform.tfvars

    db_instance_class = "db.t2.micro"
    dbname = "superherodb"
    dbuser = "superhero"
    dbpassword = "superheropass"

### Add to variables.tf

    variable "db_instance_class" {}
    variable "dbname" {}
    variable "dbuser" {}
    variable "dbpassword" {}

---

## Creating the Dev instance

### Add to main.tf
    
    #------Key pair------
    resource "aws_key_pair" "wp_auth" {
        key_name = "${var.key_name}"
        public_key = "${file(var.public_key_path)}" #References a local file 

    #------Dev Server------
    
    resource "aws_instance" "wp_dev" {
        instance_type = "${var.dev_instance_type}"
        ami = "${var.dev_ami}"    
        
        tags {
          Name = "wp_dev"
        }
        
        key_name = "${aws_key_pair.wp_auth.id}"
        vpc_security_group_ids = ["${aws_security_group.wp_dev_sg.id}"]
        iam_instance_profile = "${aws_iam_instance_profile.s3_access_profile.id}"
        subnet_id = "${aws_subnet.wp_public1_subnet.id}"
        
        provisioner "local_exec" {
          command = <<EOD
            cat <<EOF > aws_hosts
            [dev]
            ${aws_instance.wp_dev.public_ip}
            [dev:vars]
            s3code=${aws_s3_bucket.code.bucket}
            domain=${var.domain_name}
            EOF
          EOD
        }
        
        provisioner "local_exec" {
          command = "aws ec2 wait instance-status-ok --instance-ids ${aws_instance.wp_dev.id} --profile superhero && ansible-playbook -i aws_hosts wordpress.yml"
        }
    }
    
### Add to terraform.tfvars

    dev_instance_type = "t2.micro"
    dev_ami = "ami-b73b63a0" #Check on AWS console
    public_key_path = "/root/.ssh/kryptonite.pub"
    key_name = "kryptonite"
    
### Add to variables.tf

    variable "dev_instance_type" {}
    variable "dev_ami" {}
    variable "public_key_path" {}
    variable "key_name" {}

---

## Creating the ELB

### Add to main.tf

    #------Load Balncer------
    
    resource "aws_elb" "wp_elb" {
        name = "${var.domain_name}-elb"
        
        subnets = ["${aws_subnet.wp_public1_subnet.id}",
          "${aws_subnet.wp_public1_subnet.id}"
        ]
        
        security_groups = ["${aws_security_group.wp_public_sg.id}"]
        
        listener {
          instance_port = 80
          instance_protocol = "http"
          lb_port = 80
          lb_protocol = "http"
        }
        
        health_check {
          healthy_threshold = "${var.elb_healthy_threshold}"
          unhealthy_threshold = "${var.elb_unhealthy_threshold}"
          timeout = "${var.elb_timeout}"
          target = "TCP:80" #TCP instead of HTTP to avoid errors
          interval = "${var.elb_interval}"
        }
        
        cross_zone_load_balancing = true
        idle_timeout = 400
        connection_draining = true
        connection_draining_timeout = 400
        
        tags {
          Name = "wp_${var.domain_name}-elb"
        }
    }                     

### Add to terraform.tfvars

    elb_healthy_threshold = "2"
    elb_unhealthy_threshold = "2"
    elb_timeout = "3"
    elb_interval = "30"

### Add to variables.tf
              
    variable "elb_healthy_threshold" {}
    variable "elb_unhealthy_threshold" {}
    variable "elb_timeout" {}
    variable "elb_interval" {}

---

## Creating the Golden AMI

### Add to main.tf

    #------Golden AMI------
    
    #random ami id
    
    resource "random_id" "golden_ami" {
        byte_length = 3
    }
    
    # AMI
    
    resource "aws_ami_from_instance" "wp_golden" {
        name = "wp_ami-${random_id.golden_ami.b64}"
        source_instance_id = "${aws_instance.wp_dev.id}"
        
        provisioner "local-exec" {
          command = <<EOT
            cat <<EOF > userdata
            #!/bin/bash
            /usr/bin/aws s3 sync s3://${aws_s3_bucket.code.bucket} /var/www/html
            /bin/touch /var/spool/cron/root
            sudo /bin/echo '*/5 * * * * aws s3 sync s3://${aws_s3_bucket.code.bucket} /var/www/html' >> /var/spool/cron/root
            EOF
          EOT
        }
        
    }
    
---

## Configuring the auto scaling group

### Add to main.tf

    #------Launch configuration------
    
    resource "aws_launch_configuration" "wp_lc" {
        name_prefix = "wp_lc-"
        image_id = "${aws_ami_from_instance.wp_golden.id}"
        instance_type = "${var.lc_instance_type}"
        security_groups = ["${aws_security_group.wp_private_sg.id}"]
        iam_instance_profile = "${aws_iam_instance_profile.s3_access_profile.id}"
        key_name = "${aws_key_pair.wp_auth.id}"
        user_data = "${file("userdata")}"
        
        lifecycle {
          create_before_destroy = true
        }
    }
    
    #------Auto scaling group------
    
    resource "aws_autoscaling_group" "wp_asg" {
        name = "asg-${aws_launch_configuration.wp_lc.id}"
        max_size = "${var.asg_max}"
        min_size = "${var.asg_min}"
        health_check_grace_priod = "${var.asg_grace}"
        health_check_type = "${var.asg_hct}"
        desired_capacity = "${var.asg_cap}"
        force_delete = true
        load_balancers = ["${aws_elb.wp_elb.id}"]
        
        vpc_zone_identifier = ["${aws_subnet.wp_private1_subnet.id}",
          "${aws_subnet.wp_private2_subnet.id}"
        ]
        
        launch_configuration = "${aws_launch_configuration.wp_lc.name}"
        
        tag {
          key = "Name"
          value = "wp_asg-instance"
          propagate_at_launch = true
        }
        
        lifecycle {
          create_before_destroy = true
        }
    }
    
### Add to terraform.tfvars
    
    asg_max = "2"
    asg_min = "1"
    asg_grace = "300"
    asg_hct = "EC2"
    asg_cap = "2"
    lc_instance_type = "t2.micro"

### Add to variables.tf

    variable "asg_max" {}
    variable "asg_min" {}
    variable "asg_grace" {}
    variable "asg_hct" {}
    variable "asg_cap" {}
    variable "lc_instance_type" {}

---

## Creating Route 53 records

### Add to main.tf

    #------Route 53------
    
    #Primary Zone
    
    resource "aws_route53_zone" "primary" {
        name = "${var.domain_name}.com"
        delegation_set_id = "${var.delegation_set}"
    }
    
    #WWW
    
    resource "aws_route53_record" "www" {
        zone_id = "${aws_route53_zone.primary.zone_id}"
        name = "www.${var.domain_name}.com"
        type = "A"
        
        alias {
          name = "${aws_elb.wp_elb.dns_name}"
          zone_id = "${aws_elb.wp_elb.zone_id}"
          evaluate_target_health = false
        }
    }
    
    #DEV
    
    resource "aws_route53_record" "dev" {
        zone_id = "${aws_route53_zone.primary.zone_id}"
        name = "dev.${var.domain_name}.com"
        type = "A"
        ttl = "300"
        records = ["${aws_instance.wp_dev.public_ip}"]
    }
    
    #Private zone
    resource "aws_route53_record" "secondary" {
        name = "${var.domain_name}.com"
        vpc_id = "${aws_vpc.wp_vpc.id}"
    }
    
    #DB
    
    resource "aws_route53_record" "db" {
        zone_id = "${aws_route53_zone.secondary.zone_id}"
        name = "db.${var.domain_name}.com"
        type = "CNAME"
        ttl = "300"
        records = ["${aws_db_instance.wp_db_address}"]
    }
    
### Add to terraform.tfvars

    delegation_set = "N1HADZB520Q3IV" #Change it to your delegation set

### Add to variables.tf

    variable "delegation_set" {}

---

## Creating the ansible playbooks

### wordpress.yml

    ---
    - hosts: dev
      become: yes
      remote_user: ec2-user
      
      tasks:
      - name: Install Apache
          yum: 
            name={{ item }} 
            state: present    
          with_items:
          - httpd
          - php
          - php-mysql
        
      - name: Download Wordpress
          get_url: 
            url=http://wordpress.org/wordpress-latest.tar.gz
            dest=/var/www/html/wordpress.tar.gz
            force=yes
        
      - name: Extract Wordpress
          command: "tar xzf /var/ww/html/wordpress.tar.gz -C /var/www/html --strip-components 1"
        
      - name: Make directory tree readable
          file:
            path: /var/www/html
            mode: u:rwx,g:rx,o:rx
            recurse: yes
            owner:apache
            group:apache
        
      - name: Make sure Apache is started now and at boot
          service: 
            name:httpd
            state:started
            enabled:yes
            
### s3update.yml

    ---
    
    - hosts: dev
      become: yes
      remote_user: ec2-user
    
      tasks:
      - name: Update S3 code bucket
        command: aws s3 sync /var/www/html s3://{{ s3code }}/ --delete
      - shell: echo "define('WP_SITEURL','http://dev"{{ domain }}".com');" >> wp-config.php
        args:
          chdir: /var/www/html
      - shell: echo "define('WP_HOME','http://dev"{{ domain }}".com');" >> wp-config.php
        args:
          chdir: /var/www/html

---

## Terraform apply

    vi /etc/ansible/ansible.cfg
    #Change host_key_cheking = false
    
    ssh-add -l
    ssh-agent bash
    ssh-add ~/root/.ssh/kryptonite
    
    terraform apply
    
---

## Final steps

* Set the Wordpress DB values with the variables defined on terraform    
* Run installation
* Change URLs to www.
* ansible-playbook -i aws_hosts s3update.yml        
            
            
        
    
