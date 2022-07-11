---
title:  "How To Setup A Standard AWS VPC With Terraform"
date:   2019-11-25 09:00:00
permalink: how-to-setup-a-standard-aws-vpc-with-terraform
---

This is a documentation on how to setup the standard virtual private network (VPC) in AWS with the basic security configurations using Terraform.

In general, I classify the basics as having the servers and databases in the private subnets, and having a bastion server for remote access. There is definitely much room to improve from this setup and certainly much more in the realms beyond my knowledge. However, as a start, this is, at the very least, essential for a production environment,

Personally, I **had** an Amazon Certified Solutions Architect (Associate) certificate to my name, but like most of the engineering university graduates out there who have forgotten how to do `dy/dx` or  what the hell is the [L’Hôpital’s rule](https://en.wikipedia.org/wiki/L'H%C3%B4pital's_rule), I have all but forgotten the exact steps to recreate such an environment.

INSERT IMAGE AWS Associate Solutions Architect

As a saving grace 😅, I should say that I do know how to set it up, just that I do not have it at the tip of my fingers. I would not get it right the first time, but given time I will eventually set it up correctly.

This is true for whenever I setup an environment for new projects. Debugging the setup which can be time consuming and frustrating. It is not efficient and is probably one of the key reasons why [infrastructure as code (IaC)](https://en.wikipedia.org/wiki/Infrastructure_as_code) has become a trending topic in recent years.

Provisioning these infrastructures using code implies:

- version control on code and, in turn, infrastructural changes made by members of the development team
- easily reproducible infrastructures
- automation

One of the frontrunners in this industry is [Terraform](https://www.terraform.io/). All that is required are the configurations written in files ending with the “tf” extension placed in the same directory.

## The VPC
Start by provisioning the VPC.

We set the CIDR block to provide the [maximum number private ip addresses that an AWS VPC allows](https://docs.aws.amazon.com/vpc/latest/userguide/how-it-works.html#VPC_Sizing). This implies that you can have up to 65,536 AWS resources in your VPC, assuming each of them require a private IP address for communication purpose.

```terraform
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16" # 65536 ip addresses

  tags = {
    Name = "${var.project_name}${var.env}"
  }
}
```

The variables project_name and env can be placed in a separate .tf as long as they are in the same directory when Terraform eventually runs to apply the changes.

## The Gateways

Next, we setup the Internet gateway (IGW) and NAT gateway (NGW).

The IGW allows for resources in the public subnets to communicate with the outside Internet.

The NGW does the same thing,  but for the resources in the private subnets. Sometimes, these resources need to download packages from the Internet for updates etc. This is in direct conflict with the security requirements that placed them in the private subnets in the first place. The NGW balances these 2 requirements.

# IGW

```terraform
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.project_name}${var.env}"
  }
}

resource "aws_route_table" "igw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "igw-${var.project_name}${var.env}"
  }
}
```

resource "aws_route" "igw" {
  route_table_id = aws_route_table.igw.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id = aws_internet_gateway.main.id
}

# NGW

```terraform
resource "aws_route_table" "ngw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "ngw-${var.project_name}${var.env}"
  }
}

resource "aws_route" "ngw" {
  route_table_id = aws_route_table.ngw.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id = aws_nat_gateway.main.id
}

### NOTE ###
resource "aws_eip" "nat" {
  vpc = true
}

resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id = aws_subnet.public-ap-southeast-1a.id

  tags = {
    Name = "${var.project_name}${var.env}"
  }
}
```

Both gateways need to be associated to their respective aws_route_table via an aws_route that will route out to everywhere on the Internet, as indicated by the 0.0.0.0/0 CIDR block.

The NGW requires some additional setup.

First, a NAT gateway requires an elastic IP address due to the way it is engineered. I would not pretend I know how it works to tell you why a static IP address is required, but I do know we can easily provision using Terraform.

This static IP address will also come in useful if your private instances need to make API calls to third party sources that require the instances ip address for whitelisting purpose. The outgoing requests from the private instances will bear the ip address of the NGW.

In addition, a NAT gateway needs to be placed in one of the the public subnet in order to communicate with the Internet. As you can see, we have made an implicit dependency on the aws_subnet which we will define later. Terraform will ensure the NAT gateway will be created after the subnets are setup.

## The Subnets

Now, let’s setup the subnets.

We will setup 1 public and 1 private subnet in each availability zones that the region provides. I will be using the ap-southeast-1 (Singapore) region. That will be a total of 6 subnets to provision as there are 3 subnets in this region.

```terraform
#### public 1a
resource "aws_subnet" "public-ap-southeast-1a" {
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.100.0/24"
  availability_zone_id = "apse1-az2"

  tags = {
    Name = "public-ap-southeast-1a-${var.project_name}${var.env}"
  }
}

resource "aws_route_table_association" "public-ap-southeast-1a" {
  subnet_id = aws_subnet.public-ap-southeast-1a.id
  route_table_id = aws_route_table.igw.id
}

#### public 1b
resource "aws_subnet" "public-ap-southeast-1b" {
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.101.0/24"
  availability_zone_id = "apse1-az1"

  tags = {
    Name = "public-ap-southeast-1b-${var.project_name}${var.env}"
  }
}

resource "aws_route_table_association" "public-ap-southeast-1b" {
  subnet_id = aws_subnet.public-ap-southeast-1b.id
  route_table_id = aws_route_table.igw.id
}

#### public 1s
resource "aws_subnet" "public-ap-southeast-1c" {
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.102.0/24"
  availability_zone_id = "apse1-az3"

  tags = {
    Name = "public-ap-southeast-1c-${var.project_name}${var.env}"
  }
}

resource "aws_route_table_association" "public-ap-southeast-1c" {
  subnet_id = aws_subnet.public-ap-southeast-1c.id
  route_table_id = aws_route_table.igw.id
}

#### private 1a
resource "aws_subnet" "private-ap-southeast-1a" {
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
  availability_zone_id = "apse1-az2"

  tags = {
    Name = "private-ap-southeast-1a-${var.project_name}${var.env}"
  }
}

resource "aws_route_table_association" "private-ap-southeast-1a" {
  subnet_id = aws_subnet.private-ap-southeast-1a.id
  route_table_id = aws_route_table.ngw.id
}

#### private 1b
resource "aws_subnet" "private-ap-southeast-1b" {
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.2.0/24"
  availability_zone_id = "apse1-az1"

  tags = {
    Name = "private-ap-southeast-1b-${var.project_name}${var.env}"
  }
}

resource "aws_route_table_association" "private-ap-southeast-1b" {
  subnet_id = aws_subnet.private-ap-southeast-1b.id
  route_table_id = aws_route_table.ngw.id
}

#### private 1c
resource "aws_subnet" "private-ap-southeast-1c" {
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.3.0/24"
  availability_zone_id = "apse1-az3"

  tags = {
    Name = "private-ap-southeast-1c-${var.project_name}${var.env}"
  }
}

resource "aws_route_table_association" "private-ap-southeast-1c" {
  subnet_id = aws_subnet.private-ap-southeast-1c.id
  route_table_id = aws_route_table.ngw.id
}
```

Amidst this long snippet of configuration for the subnets, it is essentially a repeat of the same resources association.

For the public subnets, they are assigned the CIDR blocks `10.0.1.0/24`, `10.0.2.0/24` and `10.0.3.0/24` respectively. Each will have up to 256 ip addresses to house 256 AWS resources that requires an ip address. Their addresses will be from, taking the first subnet as example, 10.0.1.0 to 10.0.1.255.

For the private subnets, they occupy the CIDR blocks `10.0.101.0/24`, `10.0.102.0/24` and `10.0.103.0/24` respectively.

To be exact, there will be less than 256 addresses per subnet as some private IP addresses are reserved in every subnet. Of course, you can provision more or less ip addresses per subnet with the correct subnet masking setting.

Each subnet is associated to different availability zones via the availability_zone_id to spread out the resources across the region.

Each public subnet is also associated to the aws_route_table that is related to the IGW, while each private subnet is associated to the aws_route_table related to the NGW.

## The Database

Next, we setup the database. We will provision the database using RDS and place it in the private subnets for security purpose.

At this point of time, I must admit that I do not know if this is the best way to setup the database. I personally have a lot of questions on how the infrastructure will change when the application scales eventually, especially for the database. How will the database be sharded into different regions to serve a global audience? How do the database sync across the different regions? These are side quests that I will have to pursue in the future.

For now, a single instance in a private subnet.

```terraform
resource "aws_db_instance" "main" {
  allocated_storage = 20
  storage_type = "gp2"
  engine = "mysql"
  engine_version = "5.7"
  instance_class = "db.t2.micro"
  identifier = "rds-${var.project_name}${var.env}"
  name = "something"
  username = "something"
  password = "something"

  skip_final_snapshot = false
  # notes time of creation of rds.tf file
  final_snapshot_identifier = "rds-${var.project_name}${var.env}-1573454102"

  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name = aws_db_subnet_group.main.id

  lifecycle {
    prevent_destroy = true
  }

  tags = {
    Name = "rds-${var.project_name}${var.env}"
  }
}

resource "aws_db_subnet_group" "main" {
  name = "db-private-subnets"
  subnet_ids = [
    aws_subnet.private-ap-southeast-1a.id,
    aws_subnet.private-ap-southeast-1b.id,
    aws_subnet.private-ap-southeast-1c.id
  ]

  tags = {
    Name = "subnet-group-${var.project_name}${var.env}"
  }
}
```

As you can see, we can see and review the full configuration for the database using code as compared to having to navigate around the AWS management console to complete the puzzle. We can easily know the size of the database instance we have provisioned as well as its credentials (Ok this is debatable if we want to commit sensitive data in our code).

In this configuration, I ensured that the database will produce a final snap shot in the event it gets destroyed.

Access to the database will be guarded by an `aws_security_group` that will be defined later.

The database is also associated to the aws_db_subnet_group resource. This resource consist of all the private subnet that we provisioned. This creates an implicit dependency on these subnets, ensuring that the database will only be created after the subnets are created. This would also tell AWS to place the database in the custom VPC that the subnets exist in.

I also ensured the database will not be destroyed by Terraform accidentally using the lifecycle configuration.

## The Bastion

The bastion server allows us to access the servers and the database instance in the private subnets. We will provision the bastion inside the public subnet.

```terraform
resource "aws_instance" "bastion" {
  ami = "ami-061eb2b23f9f8839c"
  associate_public_ip_address = true
  instance_type = "t2.nano"
  subnet_id = aws_subnet.public-ap-southeast-1a.id
  vpc_security_group_ids = ["${aws_security_group.bastion.id}"]
  key_name = aws_key_pair.main.key_name

  tags = {
    Name = "bastion-${var.project_name}${var.env}"
  }
}

resource "aws_key_pair" "main" {
  key_name = "${var.project_name}-${var.env}"
  public_key = "ssh-rsa something"
}


output "bastion_public_ip" {
  value = aws_instance.bastion.public_ip
}
```

I am using a Ubuntu-18.04 LTS image to setup the bastion instance. Note that the AMI id will differ from region to region, even for the same operating system. The image below shows the difference in the AMI id between Singapore and Tokyo regions.

INSERT IMAGE ubuntu ami in ap-southeast-1 vs ubuntu ami in tokyo region

I will mainly use the bastion to tunnel the commands to the private subnet. Hence, there is no need for a large computation. The cheapest and smallest instance size of `t2.nano` is chosen.

It is associated to a public subnet that we created. Any subnet will work, but make sure it is public as we need to be able to connect to it.

Its security group will be defined later.

All EC2 instances in AWS can be given an `aws_key_pair`. We can generate a custom private key using the ssh-keygen command or you can use the default ssh key in your local machine so that you can ssh into the bastion easily without having to define the identity file each time you do so.

Then, there is the output block. After Terraform has completed its magic, it will output values defined in these output blocks. In this case, the public ip address of the bastion server will be shown on the terminal, making it easy for us to obtain the endpoint.

## The Security Groups

Lastly, the connection is not completed without setting up the security groups that guards the traffic going in and out of the resources. This was the bane of my AWS Solution Architect journey. With the required configurations spelled out in code instead of steps in the console that exist only in the memory, Terraform has helped me greatly to further understand this feature.

There are a total of 3 aws_security_group resources  to be created, representing the bastion, the instances and the database respectively. Each of them have their own set of inbound and/or outbound rules, named “ingress” and “egress” in Terraform terms, that are configured separately.

While you can configure the inbound and outbound rules together within the resource block of the respective aws_security_group, I would recommend against that. This is because doing so will result in tight coupling between the security groups, especially if one of its aws_security_group_rule is pointing to another aws_security_group as the source. This is problematic when we eventually make changes to the security groups because, for example, maybe one cannot be destroyed because a security group that is it dependent on is not supposed to be destroyed.

And the frustrating thing is that `Terraform`, or maybe the underlying AWS api, do not indicate the error. In fact it takes forever to destroy security groups that are created this way, only to fail after making us wait for a long time, which makes debugging superfluously tedious.

There are many issues mentioning this and something related on Github, like this. This has to do with has been termed “enforced dependencies” that Terraform currently has no mechanism to handle.

By decoupling the `aws_security_group` and their respective `aws_security_group_rule` into separate resources, we will give `Terraform` and ourselves an easier time removing and making changes to the security groups in the future.

## Bastion

Let’s see how we can configure `Terraform` setup the security of the subnets. We start off with the security group for the bastion server. We will make 3 rules for it.

```terraform
# bastion
resource "aws_security_group" "bastion" {
  name = "${var.project_name}${var.env}-bastion"
  description = "For bastion server ${var.env}"
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.project_name}${var.env}"
  }
}

resource "aws_security_group_rule" "ssh-bastion-world" {
  type = "ingress"
  from_port = 22
  to_port = 22
  protocol = "tcp"
  # Please restrict your ingress to only necessary IPs and ports.
  # Opening to 0.0.0.0/0 can lead to security vulnerabilities
  # You may want to set a fixed ip address if you have a static ip
  security_group_id = aws_security_group.bastion.id
  cidr_blocks = ["0.0.0.0/0"]
}

resource "aws_security_group_rule" "ssh-bastion-web_server" {
  type = "egress"
  from_port = 22
  to_port = 22
  protocol = "tcp"
  security_group_id = aws_security_group.bastion.id
  source_security_group_id = aws_security_group.web_server.id
}

resource "aws_security_group_rule" "mysql-bastion-rds" {
  type = "egress"
  from_port = 3306
  to_port = 3306
  protocol = "tcp"
  security_group_id = aws_security_group.bastion.id
  source_security_group_id = aws_security_group.rds.id
}
```

The first is an ingress rule to allow us to ssh into it from wherever we are. Of course, this is not ideal as it means anyone from anywhere can ssh into it. We should scope it to the ip address where you work from, be it your home or your office. However, for my case, as a digital nomad, the ip address that I work with just changes so often as I moved around that it just makes more sense to open it up to the world. I made a calculated risk here. Please don’t try this at home.

The second is an egress rule that allow the bastion instance to ssh into the web servers in the private subnets. The source of this rule is set as the aws_security_group of the web servers.

The third rule is another outbound rule  to allow the bastion to communicate with the database. Since I am using <code>mysql</code> as the database engine, the port used is 3306. This allows us to run database operation on the isolated database instance in the private subnet via the bastion over the correct port securely.

## Web Servers

Next will be the security groups for your web servers. The only rule that it requires will be the ingress rule for the bastion to ssh into itself over port 22.

```terraform
resource "aws_security_group" "web_server" {
  name = "${var.project_name}${var.env}-web-servers"
  description = "For Web servers ${var.env}"
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.project_name}${var.env}"
  }
}

resource "aws_security_group_rule" "ssh-web_server-bastion" {
  type = "ingress"
  from_port = 22
  to_port = 22
  protocol = "tcp"
  security_group_id = aws_security_group.web_server.id
  source_security_group_id = aws_security_group.bastion.id
}
RDS
Lastly, the rds instance. It consist of 2 rules.

resource "aws_security_group" "rds" {
    name = "rds-${var.project_name}${var.env}"
    description = "For RDS ${var.env}"

vpc_id = aws_vpc.main.id
  tags = {
    Name = "${var.project_name}${var.env}"
  }
}

resource "aws_security_group_rule" "mysql-rds-web_server" {
  type = "ingress"
  from_port = 3306
  to_port = 3306
  protocol = "tcp"
  security_group_id = aws_security_group.rds.id
  source_security_group_id = aws_security_group.web_server.id
}

resource "aws_security_group_rule" "mysql-rds-bastion" {
  type = "ingress"
  from_port = 3306
  to_port = 3306
  protocol = "tcp"
  security_group_id = aws_security_group.rds.id
  source_security_group_id = aws_security_group.bastion.id
}
```

The first is of course to open up port 3306 to allow request from the web servers to reach the database to run the application.

The second is to allow the bastion to communicate over port 3306. We have to define the egress rule applied on the bastion server itself to connect out to the RDS instance previously. Now, this ingress rule will allow the incoming request from the bastion server to reach the RDS instance instead of being blocked off.

## Terraform Apply

These resources can be defined in a single or multiple terraform files with the extension tf, as long as they are in the same directory.

If you are [using docker to run terraform](https://vic-l.github.io/terraform-with-docker), you can do a volume mount of the current directory into the workspace of the docker container and apply the infrastructure!

## Improvements

We can harden the security of this setup further by, for example, configuring the [Network Access Control Level (NACL or Network ACL)](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html). In this setup, the default is allow all traffic in bound and outbound for all the resources. However, this will be beyond the scope of this article.

## What’s Next

Note that I did not provision any EC2 instances where my application will run. At this point of time, you can feel free to provision the EC2 instances for the web servers just like the bastion server, but associating them with the private subnets.

For me, I favor [AWS Elastic Beanstalk](https://aws.amazon.com/elasticbeanstalk/) in handling the deployment. What I have done so far is only the provisioning of the infrastructure. Hence, in my case, instead of defining the EC2 instances, I will define an [elastic beanstalk environment](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/elastic_beanstalk_environment.html) to host my Rails application and configure it to use the VPC to leverage on all the security.
