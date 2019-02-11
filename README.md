# Setup a Container Cluster on AWS with Terraform

![banner](https://s3-ap-southeast-2.amazonaws.com/aws-ecs-setup/ecs_banner.png)

## A demo of how to configure a container cluster on AWS and run a service on the cluster.

Breakdown of tasks:

 - Provision your VPC, subnet, routing table and security group.
 - Set up IAM roles and instance profile.
 - Set up an ALB.
 - Set up an Autoscaling group and launch configuration.
 - Create the ECS cluster.
 - Create the Task definition and service to run on the ECS cluster.

### VPC, subnet, routing table and security group setup:

 - Navigate to/create a terraform directory on your machine. (I am using a win 10 machine with powershell FYI)

 ```
 PS C:\Program Files\terraform\configs\ecs_setup_files>
 ```

 - Create a file called 'vpc.tf', which contains the following configuration:

 ```
   # Define a vpc
  resource "aws_vpc" "demoVPC" {
    cidr_block = "200.0.0.0/16"
    tags {
      Name = "ecsDemoVPC"
    }
  }
 ```

 - Next, you will need to create an internet gateway, in order for the VPC to communicate with the internet.

 ```
   # Internet gateway for the public subnet
  resource "aws_internet_gateway" "demoIG" {
    vpc_id = "${aws_vpc.demoVPC.id}"
    tags {
      Name = "ecsDemoIG"
    }
  }
 ```

 - The above internet gateway must be linked to the VPC resource like this: "vpc_id = "${aws_vpc.demoVPC.id}". Terrafrom will always provision the vpc before provisioning the internet gateway.
 - All container instances will need to reside in a public subnet, to which they will be registered. The following resource takes care of this:

 ```
   # Public subnet
  resource "aws_subnet" "demoPubSN0-0" {
    vpc_id = "${aws_vpc.demoVPC.id}"
    cidr_block = "200.0.0.0/24"
    availability_zone = "us-east-1a"
    tags {
      Name = "ecsDemoPubSN0-0-0"
    }
  }
 ```

 - Similar to internet gateway, I have linked the resource to the VPC. Next, configure the routing table for the subnet:

 ```
   # Routing table for public subnet
  resource "aws_route_table" "demoPubSN0-0RT" {
    vpc_id = "${aws_vpc.demoVPC.id}"
    route {
      cidr_block = "0.0.0.0/0"
      gateway_id = "${aws_internet_gateway.demoIG.id}"
    }
    tags {
      Name = "demoPubSN0-0RT"
    }
  }
 ```

 - Again, add the VPC id to the resource. I have also referenced the internet gateway for the resource. This route directs all traffic to the internet.
 - The final step in setting up the VPC, involves associating the routing table to the public subnet:

 ```
   # Associate the routing table to public subnet
  resource "aws_route_table_association" "demoPubSN0-0RTAssn" {
    subnet_id = "${aws_subnet.demoPubSN0-0.id}"
    route_table_id = "${aws_route_table.demoPubSN0-0RT.id}"
  }
 ```

  - The final step requires the addition of a security group to determine what goes in and what goes out of your VPC:

 ```
   # ECS Instance Security group

  resource "aws_security_group" "test_public_sg" {
      name = "test_public_sg"
      description = "Test public access security group"
      vpc_id = "${aws_vpc.test_vpc.id}"

     ingress {
         from_port = 22
         to_port = 22
         protocol = "tcp"
         cidr_blocks = [
            "0.0.0.0/0"]
     }

     ingress {
        from_port = 80
        to_port = 80
        protocol = "tcp"
        cidr_blocks = [
            "0.0.0.0/0"]
     }

     ingress {
        from_port = 8080
        to_port = 8080
        protocol = "tcp"
        cidr_blocks = [
            "0.0.0.0/0"]
      }

     ingress {
        from_port = 0
        to_port = 0
        protocol = "tcp"
        cidr_blocks = [
           "${var.test_public_01_cidr}",
           "${var.test_public_02_cidr}"]
      }

      egress {
          # allow all traffic to private SN
          from_port = "0"
          to_port = "0"
          protocol = "-1"
          cidr_blocks = [
              "0.0.0.0/0"]
      }

      tags {
         Name = "test_public_sg"
       }
  }
 ```
