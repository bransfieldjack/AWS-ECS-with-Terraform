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

### 1. (VPC, subnet, routing table and security group setup):

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

 - You should end up with the following:

 ![expected_output](https://s3-ap-southeast-2.amazonaws.com/aws-ecs-setup/expected_output.PNG)
 ![verified](https://s3-ap-southeast-2.amazonaws.com/aws-ecs-setup/vpc_created.PNG)

 - Once verified, you can destroy the resources using the ```terraform destroy``` command.

### 2. (Variable definition, IAM roles, load balancers, auto scaling groups and launch configuration, ECS cluster, task definitions and service creation):

 - Terraform can retrieve variables from input files. You can create your own variables input file (variable.tf). Below is some boiler plate input for a variables file.

 ```
   # main creds for AWS connection
  variable "aws_access_key_id" {
    description = "AWS access key"
  }

  variable "aws_secret_access_key" {
    description = "AWS secret access key"
  }

  variable "ecs_cluster" {
    description = "ECS cluster name"
  }

  variable "ecs_key_pair_name" {
    description = "EC2 instance key pair name"
  }

  variable "region" {
    description = "AWS region"
  }

  variable "availability_zone" {
    description = "availability zone used for the demo, based on region"
    default = {
      us-east-1 = "us-east-1"
    }
  }

  ########################### Test VPC Config ################################

  variable "test_vpc" {
    description = "VPC name for Test environment"
  }

  variable "test_network_cidr" {
    description = "IP addressing for Test Network"
  }

  variable "test_public_01_cidr" {
    description = "Public 0.0 CIDR for externally accessible subnet"
  }

  variable "test_public_02_cidr" {
    description = "Public 0.0 CIDR for externally accessible subnet"
  }

  ########################### Autoscale Config ################################

  variable "max_instance_size" {
    description = "Maximum number of instances in the cluster"
  }

  variable "min_instance_size" {
    description = "Minimum number of instances in the cluster"
  }

  variable "desired_capacity" {
    description = "Desired number of instances in the cluster"
  }
 ```

  - If you wish, you can specify the values for each of the variables in a ```terraform.tfvars``` file. This is similar to having an environment variable and I would recommend that if you do choose to use this, that you add the file to your .gitignore file before pushing any changes to your publicly visible repo.

  ```
    ecs_cluster="$ECS_CLUSTER"
    ecs_key_pair_name="$ECS_KEY_PAIR_NAME"
    aws_access_key_id = "$PECT_AWS_KEYS_INTEGRATION_ACCESSKEY"
    aws_secret_access_key = "$PECT_AWS_KEYS_INTEGRATION_SECRETKEY"
    region = "$REGION"
    test_vpc = "$TEST_VPC"
    test_network_cidr = "$TEST_NETWORK_CIDR"
    test_public_01_cidr = "$TEST_PUBLIC_01_CIDR"
    test_public_02_cidr = "$TEST_PUBLIC_02_CIDR"
    max_instance_size = "$MAX_INSTANCE_SIZE"
    min_instance_size = "$MIN_INSTANCE_SIZE"
    desired_capacity = "$DESIRED_CAPACITY"
  ```

  - The next step will be to setup the IAM roles for the cluster. IAM roles are required for the container agent to work. The container agent is required for calls to the API etc. and the containers will not work without it. They are also required for use with the ECS Service Scheduler which communicates with the load balancers. We can also use an instance profile to pass role information to the clusters EC2 instances.

### ecs-service-role.tf

  ```
    resource "aws_iam_role" "ecs-service-role" {
      name                = "ecs-service-role"
      path                = "/"
      assume_role_policy  = "${data.aws_iam_policy_document.ecs-service-policy.json}"
  }

  resource "aws_iam_role_policy_attachment" "ecs-service-role-attachment" {
      role       = "${aws_iam_role.ecs-service-role.name}"
      policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceRole"
  }

  data "aws_iam_policy_document" "ecs-service-policy" {
      statement {
          actions = ["sts:AssumeRole"]

          principals {
              type        = "Service"
              identifiers = ["ecs.amazonaws.com"]
          }
      }
  }
  ```

### ecs-instance-role.tf

  ```
    resource "aws_iam_role" "ecs-instance-role" {
        name                = "ecs-instance-role"
        path                = "/"
        assume_role_policy  = "${data.aws_iam_policy_document.ecs-instance-policy.json}"
    }

    data "aws_iam_policy_document" "ecs-instance-policy" {
        statement {
            actions = ["sts:AssumeRole"]

            principals {
                type        = "Service"
                identifiers = ["ec2.amazonaws.com"]
            }
        }
    }

    resource "aws_iam_role_policy_attachment" "ecs-instance-role-attachment" {
        role       = "${aws_iam_role.ecs-instance-role.name}"
        policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role"
    }

    resource "aws_iam_instance_profile" "ecs-instance-profile" {
        name = "ecs-instance-profile"
        path = "/"
        roles = ["${aws_iam_role.ecs-instance-role.id}"]
        provisioner "local-exec" {
          command = "sleep 10"
        }
    }
  ```

   - Next we will have to setup a load balancer to balance the traffic between instances.

  ```
    resource "aws_alb" "ecs-load-balancer" {
      name                = "ecs-load-balancer"
      security_groups     = ["${aws_security_group.test_public_sg.id}"]
      subnets             = ["${aws_subnet.test_public_sn_01.id}", "${aws_subnet.test_public_sn_02.id}"]

      tags {
        Name = "ecs-load-balancer"
      }
  }

  resource "aws_alb_target_group" "ecs-target-group" {
      name                = "ecs-target-group"
      port                = "80"
      protocol            = "HTTP"
      vpc_id              = "${aws_vpc.test_vpc.id}"

      health_check {
          healthy_threshold   = "5"
          unhealthy_threshold = "2"
          interval            = "30"
          matcher             = "200"
          path                = "/"
          port                = "traffic-port"
          protocol            = "HTTP"
          timeout             = "5"
      }

      tags {
        Name = "ecs-target-group"
      }
  }

  resource "aws_alb_listener" "alb-listener" {
      load_balancer_arn = "${aws_alb.ecs-load-balancer.arn}"
      port              = "80"
      protocol          = "HTTP"

      default_action {
          target_group_arn = "${aws_alb_target_group.ecs-target-group.arn}"
          type             = "forward"
      }
  }
  ```
### Auto scaling and launch configuration

 - A launch configuration specifies the instance template that is used by an Autoscaling group to launch EC2 instances. This should be customized based on the requirements of your application.

### launch-configuration.tf

 ```
   resource "aws_launch_configuration" "ecs-launch-configuration" {
      name                        = "ecs-launch-configuration"
      image_id                    = "ami-fad25980"
      instance_type               = "t2.xlarge"
      iam_instance_profile        = "${aws_iam_instance_profile.ecs-instance-profile.id}"

      root_block_device {
        volume_type = "standard"
        volume_size = 100
        delete_on_termination = true
      }

      lifecycle {
        create_before_destroy = true
      }

      security_groups             = ["${aws_security_group.test_public_sg.id}"]
      associate_public_ip_address = "true"
      key_name                    = "${var.ecs_key_pair_name}"
      user_data                   = <<EOF
                                    #!/bin/bash
                                    echo ECS_CLUSTER=${var.ecs_cluster} >> /etc/ecs/ecs.config
                                    EOF
  }
 ```

### autoscaling-group.tf

 ```
   resource "aws_autoscaling_group" "ecs-autoscaling-group" {
      name                        = "ecs-autoscaling-group"
      max_size                    = "${var.max_instance_size}"
      min_size                    = "${var.min_instance_size}"
      desired_capacity            = "${var.desired_capacity}"
      vpc_zone_identifier         = ["${aws_subnet.test_public_sn_01.id}", "${aws_subnet.test_public_sn_02.id}"]
      launch_configuration        = "${aws_launch_configuration.ecs-launch-configuration.name}"
      health_check_type           = "ELB"
    }
 ```

- Next we will create the ECS cluster.

### cluster.tf

 ```
   resource "aws_ecs_cluster" "test-ecs-cluster" {
       name = "${var.ecs_cluster}"
   }
 ```

 - Now we can create the task definition for the cluster. A task definition defines the containers that run on the ECS cluster. Here we define two containers required to run WordPress, which are linked together along with their CPU and memory requirements.

 ```
   data "aws_ecs_task_definition" "wordpress" {
    task_definition = "${aws_ecs_task_definition.wordpress.family}"
  }

  resource "aws_ecs_task_definition" "wordpress" {
      family                = "hello_world"
      container_definitions = <<DEFINITION
  [
    {
      "name": "wordpress",
      "links": [
        "mysql"
      ],
      "image": "wordpress",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 80,
          "hostPort": 80
        }
      ],
      "memory": 500,
      "cpu": 10
    },
    {
      "environment": [
        {
          "name": "MYSQL_ROOT_PASSWORD",
          "value": "password"
        }
      ],
      "name": "mysql",
      "image": "mysql",
      "cpu": 10,
      "memory": 500,
      "essential": true
    }
  ]
  DEFINITION
  }
 ```

 - Now we can create a service. A service specified the desired number of instances (replicas) of the task definition to run on the cluster. The load balancer distrubutes traffic across all the instances of the task. The ECS Service scheduler automatically creates new task instances should any instance terminate unexpectedly.

 ```
   resource "aws_ecs_service" "test-ecs-service" {
    	name            = "test-ecs-service"
    	iam_role        = "${aws_iam_role.ecs-service-role.name}"
    	cluster         = "${aws_ecs_cluster.test-ecs-cluster.id}"
    	task_definition = "${aws_ecs_task_definition.wordpress.family}:${max("${aws_ecs_task_definition.wordpress.revision}", "${data.aws_ecs_task_definition.wordpress.revision}")}"
    	desired_count   = 2

    	load_balancer {
      	target_group_arn  = "${aws_alb_target_group.ecs-target-group.arn}"
      	container_port    = 80
      	container_name    = "wordpress"
  	}
  }
 ```
