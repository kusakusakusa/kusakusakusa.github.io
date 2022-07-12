---
title:  "How To SSH Into Private Servers Via A Bastion Without Copying SSH Keys"
date:   2020-04-27 09:00:00
permalink: how-to-ssh-into-private-servers-via-a-bastion-without-copying-ssh-keys
---

This is a documentation on the the process of accessing the public EC2 instances from a bastion server that is created in the private subnet, as a follow up to the [article on setting up a proper cloud infrastructure with basic security for applications on AWS](https://vic-l.github.io/how-to-setup-a-standard-aws-vpc-with-terraform).

## Motivation

Once in a while, we need to communicate with the production servers to do checks. The proper way is to setup a bastion server in the public instance and ssh into them. However, setting the bastion server up with the proper configurations might be time consuming to get it right.

On top of that, once we are done, there is a financial incentive to shut the bastion server down to save cost. This will translate to more time consumed to spin it up and down.

Since we might not do this often, we would tend to forget how to set up or shut down the bastion server properly. This translates to more time debugging during each process should any steps be missed along the way.

Hence, it will be nice to have these processes recorded down in code.

## Terraform Setup For Bastion Server

The terraform files to setup the bastion server is as shown below. This is a complete copy of the snippet in the [article on setting up a standard AWS VPC using terraform](https://vic-l.github.io/how-to-setup-a-standard-aws-vpc-with-terraform). The explanation is there so I would not be covering that here.

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

## Retrieving AWS EC2 Client

I will be using ruby to carry out the ssh process because I am really bad with shell script 🙁

These snippets here are translated from a rake task which is what I use for my projects. There may be errors here and there so do understand the process rather than just plain copy!

```ruby
require 'aws-sdk'

aws_access_key_id = `aws --profile #{aws_profile} configure get aws_access_key_id`.chomp
aws_secret_access_key = `aws --profile #{aws_profile} configure get aws_secret_access_key`.chomp

ec2_client = Aws::EC2::Client.new(
  region: region,
  access_key_id: aws_access_key_id,
  secret_access_key: aws_secret_access_key
)
```

First, initialize an instance of EC2 client. That will require the correct access key id and secret access key. These can be easily retrieve by running the aws configure command in the shell.

Line 3 and 4 does this, given the desired aws_profile, and stores the value in respective ruby variables for use in the rest of the script.

Note that the back ticks (`), among other shell execution commands in ruby, should be used here as it is the only one the returns the output that we need to use. The chomp method removes the line break that is returned along with the output in the shell.

## Retrieving The Bastion Server Instance

```ruby
results = ec2_client.describe_instances(
  filters: [
    {
      name: 'instance.group-name',
      values: ["#{project_name}#{env}-bastion"]
    }
  ]
)

raise 'There are more than 1 reservations. Please check!' if results.reservations.count > 1

raise 'There are no reservations. Please check!') if results.reservations.count.zero?

instances = results.reservations.first.instances

raise 'There are more than 1 bastion servers. Please check!' if instances.count > 1

bastion = instances.first
```

Next we retrieve the bastion instance via the describe_instances method as shown on line 1.

On line 4 and 5, we narrow down our instances to search for using the filter instance.group-name. This filter refers to the security group that we have attached to the bastion instance (see the terraform files).

The next few lines handle the unexpected scenarios of having multiple reservations and instances. I do not know the difference between these 2 entities, but I guess that is trivial to our mission here.

Eventually, we will have the access to the bastion instance.

## Retrieving The Private Web Server Instance

```ruby
results = ec2_client.describe_instances(
  filters: [
    {
      name: 'instance.group-name',
      values: ["#{project_name}#{env}-web-servers"]
    }
  ]
)

raise 'There are no reservations' if results.reservations.count.zero?

private_ip_addresses = results.reservations.map do |reservation|
  reservation.instances.map(&:private_ip_address)
end.flatten

raise 'There are no private_ip_addresses.' if private_ip_addresses.count.zero?

instance_ip = private_ip_addresses.first
```

Next, we retrieve web server instances using the same method by filtering their security group’s name. Our target here is the private ip address of any one of the instances. Adjust accordingly if there is a particular private instance you are trying to access.

## SSH Into Private Web Server Via Bastion Server

Here comes the main event, the ssh operation.

```ruby
sh 'ssh-add -D'
sh "ssh-add -K #{Rails.root}/#{project_name}-#{env}"
sh('ssh ' \
'-tt ' \
'-A ' \
"-i #{Rails.root}/#{project_name}-#{env} " \
"ec2-user@#{bastion.public_ip_address} " \
'-o StrictHostKeyChecking=no ' \
"-o 'UserKnownHostsFile /dev/null' " \
"\"ssh ec2-user@#{instance_ip} " \
"-o 'UserKnownHostsFile /dev/null' " \
'-o StrictHostKeyChecking=no"')
sh 'ssh-add -D'
```

Use the sh utility command in ruby to execute shell script. We are not going to use the output of the commands here, so using back ticks is not necessary.

In this bash session, line 1 clears all ssh identities present if any with the -D option. Note that this bash session is decoupled from the bash session of the current terminal. Hence, at this point of time, there should not be any since it is a new session. We also do not need to worry about erasing the ssh agents that we have added. I am keeping it here for hygiene sake.

Line 2 adds the RSA identity of the key pair, which is used to create the bastion instance, to the ssh agent in the current session.

This step is extremely pivotal. It allows us to forward our ssh agent along with the required RSA identity to the bastion server. The bastion server will subsequently be able to authenticate with the web server due to the forwarded identity.

And realise this. All this is done without the bastion server actually possessing the ssh keys at all! This is immensely beneficial on the security side of things because the bastion server, as a server on the public subnet that is exposed to the Internet, is a point of vulnerability for your private instances. It poses a security risk if attackers are able to access the private instances using the ssh keys in the bastion server. But since the keys are not there, we can make sure Gandalf sees to them.

>> INSERT MEME of Gandalf

The command from line 3 onwards is the actual main ssh command.

Line 4 forces a [pseudo-terminal](https://en.wikipedia.org/wiki/Pseudoterminal) allocation for us to interact with the web server once we have established the connection. Multiple t option ensures that the interactive session will be forced even if the ssh did not have a local [tty](https://en.wikipedia.org/wiki/Tty_(unix)) for interaction purpose.

Line 5 forwards the ssh agent through the tunneling. And since we have added the RSA identity to the our ssh agent in this session, the authentication keys are also forwarded in the process, without make a copy in the bastion server itself.

Line 6 points to the identity file required to access the bastion server from our local machine. You would not need this line if you have created your bastion server and your web instances using the same key pair. For this case, this is an extra step that is not necessary as I have set up the web and bastion servers to use the same key pair.

Line 7 states the endpoint of the bastion server and where to ssh into.

Line 8 prevents the ssh mechanism to ask for our confirmation to carry out the operation.

Line 9 prevents our bastion’s ip address to be registered a known host on our ssh known_hosts file. As these bastion servers are meant to be shut down after use, their ip addresses will be different each time we spin them up. Hence, this option will prevent the unnecessary and unmonitored bulging of our known_hosts file.

Line 10 is the command to run after we have successfully ssh into the bastion server. In this case, we are running the a subsequent ssh command to ssh into the web server instance via its private ip address that we found earlier. Note that this command is wrapped in quotes.

To reiterate a few pointers here:

- This subsequent ssh command does not require any identity file due to the RSA identity that is forwarded
- The bastion server can access the private instances due to the setup of the security groups
- The shell interaction session between the bastion server and the web instance is available to use on our local machine due to the -tt option mentioned in line 4.

Line 11 and 12 serve the same purpose as line 8 and, but this time for the bastion server’s ssh operation into the web servers.

Line 13 for hygiene sake, clear the RSA identity from the ssh agent.

## Conclusion

There you have it, an ssh session via a bastion server without copying the security keys into it to ensure minimum vulnerability and maximum security!

