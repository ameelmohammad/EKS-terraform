# EKS-terraform

I have created a blog detailing the process of setting up an Amazon Elastic Kubernetes Service (EKS) cluster with one master node and two worker nodes using Terraform. We focus on ensuring secure inbound traffic for the cluster.

The blog starts with defining the necessary provider details in the Terraform script to connect to the correct AWS region. Then, we proceed to create a custom Virtual Private Cloud (VPC) with designated CIDR blocks for a controlled environment. Within the VPC, we establish two subnets in separate Availability Zones for high availability and fault tolerance.

To allow secure inbound traffic to the cluster, we set up a security group. This security group acts as a virtual firewall, permitting only specific types of inbound traffic based on defined rules. By configuring the security group properly, we can control access to the cluster’s resources and ensure that only authorized traffic is allowed.

Next, creating IAM roles for the EKS cluster, enabling seamless interactions with other AWS services while maintaining security and access control.

Finally creating the EKS cluster itself. We delve into creating one master node that handles cluster control and two worker nodes responsible for running the actual workloads. These worker nodes are represented as EC2 instances, dynamically scaling to match workload demands.

General Setup
Setup an AWS EC2 Instance

Log in to an AWS account using a user with admin privileges and ensure your region is set to ap-northeast-2

Move to the EC2 console. Click Launch Instance.


Connect to the instance and install terraform

Installing Terraform
sudo apt update

sudo apt-get install -y gnupg software-properties-common

wget -O- https://apt.releases.hashicorp.com/gpg | \
gpg --dearmor | \
sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update

sudo apt install terraform
After running the above command we can verify by using the command

terraform --version

Creating IAM User and attach policy as “adminfullaccess” for this demo and create access key and secret key .



Creating a Terraform Script for the EKS Cluster
Launch EC2 Instance
Created a folder eks-demo and inside that created a eks.tf file . If the AWS CLI isn’t installed then you may face some issues so the best way to perform you just to install the CLI first and then execute further.
Past this script in eks.tf

#Adding Provider details
provider "aws" {
    region = "ap-south-1"
    access_key = "YOUR ACCESS KEY"
    secret_key = "YOUR SECRET KEY"
  }

#Create a custom VPC
  resource "aws_vpc" "myvpc" {
    cidr_block = "10.0.0.0/16"
    tags = {
      "Name" = "MyProjectVPC"
    }
  }

#Create Subnets
  resource "aws_subnet" "Mysubnet01" {
    vpc_id                  = aws_vpc.myvpc.id
    cidr_block              = "10.0.1.0/24"
    availability_zone       = "ap-south-1a"
    map_public_ip_on_launch = true
    tags = {
      "Name" = "MyPublicSubnet01"
    }
  }

  resource "aws_subnet" "Mysubnet02" {
    vpc_id                  = aws_vpc.myvpc.id
    cidr_block              = "10.0.2.0/24"
    availability_zone       = "ap-south-1b"
    map_public_ip_on_launch = true
    tags = {
      "Name" = "MyPublicSubnet02"
    }
  }

  # Creating Internet Gateway IGW
  resource "aws_internet_gateway" "myigw" {
    vpc_id = aws_vpc.myvpc.id
    tags = {
      "Name" = "MyIGW"
    }
  }

  # Creating Route Table
  resource "aws_route_table" "myroutetable" {
    vpc_id = aws_vpc.myvpc.id
    tags = {
      "Name" = "MyPublicRouteTable"
    }
  }

  # Create a Route in the Route Table with a route to IGW
  resource "aws_route" "myigw_route" {
    route_table_id         = aws_route_table.myroutetable.id
    destination_cidr_block = "0.0.0.0/0"
    gateway_id             = aws_internet_gateway.myigw.id
  }

  # Associate Subnets with the Route Table
  resource "aws_route_table_association" "Mysubnet01_association" {
    route_table_id = aws_route_table.myroutetable.id
    subnet_id      = aws_subnet.Mysubnet01.id
  }

  resource "aws_route_table_association" "Mysubnet02_association" {
    route_table_id = aws_route_table.myroutetable.id
    subnet_id      = aws_subnet.Mysubnet02.id
  }


 # Adding security group
  resource "aws_security_group" "allow_tls" {
    name_prefix   = "allow_tls_"
    description   = "Allow TLS inbound traffic"
    vpc_id        = aws_vpc.myvpc.id

    ingress {
      description = "TLS from VPC"
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }

    egress {
      from_port   = 0
      to_port     = 0
      protocol    = "-1"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }

 # Creating IAM role for EKS
  resource "aws_iam_role" "master" {
    name = "ed-eks-master"

    assume_role_policy = jsonencode({
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {
            "Service": "eks.amazonaws.com"
          },
          "Action": "sts:AssumeRole"
        }
      ]
    })
  }

  resource "aws_iam_role_policy_attachment" "AmazonEKSClusterPolicy" {
    policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
    role       = aws_iam_role.master.name
  }

  resource "aws_iam_role_policy_attachment" "AmazonEKSServicePolicy" {
    policy_arn = "arn:aws:iam::aws:policy/AmazonEKSServicePolicy"
    role       = aws_iam_role.master.name
  }

  resource "aws_iam_role_policy_attachment" "AmazonEKSVPCResourceController" {
    policy_arn = "arn:aws:iam::aws:policy/AmazonEKSVPCResourceController"
    role       = aws_iam_role.master.name
  }

  resource "aws_iam_role" "worker" {
    name = "ed-eks-worker"

    assume_role_policy = jsonencode({
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {
            "Service": "ec2.amazonaws.com"
          },
          "Action": "sts:AssumeRole"
        }
      ]
    })
  }

  resource "aws_iam_policy" "autoscaler" {
    name = "ed-eks-autoscaler-policy"
    policy = jsonencode({
      "Version": "2012-10-17",
      "Statement": [
        {
          "Action": [
            "autoscaling:DescribeAutoScalingGroups",
            "autoscaling:DescribeAutoScalingInstances",
            "autoscaling:DescribeTags",
            "autoscaling:DescribeLaunchConfigurations",
            "autoscaling:SetDesiredCapacity",
            "autoscaling:TerminateInstanceInAutoScalingGroup",
            "ec2:DescribeLaunchTemplateVersions"
          ],
          "Effect": "Allow",
          "Resource": "*"
        }
      ]
    })
  }

  resource "aws_iam_role_policy_attachment" "AmazonEKSWorkerNodePolicy" {
    policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
    role       = aws_iam_role.worker.name
  }

  resource "aws_iam_role_policy_attachment" "AmazonEKS_CNI_Policy" {
    policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
    role       = aws_iam_role.worker.name
  }

  resource "aws_iam_role_policy_attachment" "AmazonSSMManagedInstanceCore" {
    policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
    role       = aws_iam_role.worker.name
  }

  resource "aws_iam_role_policy_attachment" "AmazonEC2ContainerRegistryReadOnly" {
    policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
    role       = aws_iam_role.worker.name
  }

  resource "aws_iam_role_policy_attachment" "x-ray" {
    policy_arn = "arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess"
    role       = aws_iam_role.worker.name
  }

  resource "aws_iam_role_policy_attachment" "s3" {
    policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
    role       = aws_iam_role.worker.name
  }

  resource "aws_iam_role_policy_attachment" "autoscaler" {
    policy_arn = aws_iam_policy.autoscaler.arn
    role       = aws_iam_role.worker.name
  }

  resource "aws_iam_instance_profile" "worker" {
    depends_on = [aws_iam_role.worker]
    name       = "ed-eks-worker-new-profile"
    role       = aws_iam_role.worker.name
  }

 # Creating EKS Cluster
  resource "aws_eks_cluster" "eks" {
    name     = "pc-eks"
    role_arn = aws_iam_role.master.arn

    vpc_config {
      subnet_ids = [aws_subnet.Mysubnet01.id, aws_subnet.Mysubnet02.id]
    }

    tags = {
      "Name" = "MyEKS"
    }

    depends_on = [
      aws_iam_role_policy_attachment.AmazonEKSClusterPolicy,
      aws_iam_role_policy_attachment.AmazonEKSServicePolicy,
      aws_iam_role_policy_attachment.AmazonEKSVPCResourceController,
    ]
  }
# instance
  resource "aws_instance" "kubectl-server" {
    ami                         = "ami-0f5ee92e2d63afc18"  # Replace with a valid AMI ID for ap-south-1 region
    key_name                    = "mumbai-kp"  # Make sure the key pair exists in ap-south-1 region
    instance_type               = "t2.micro"
    associate_public_ip_address = true
    subnet_id                   = aws_subnet.Mysubnet01.id
    vpc_security_group_ids      = [aws_security_group.allow_tls.id]

    tags = {
      Name = "kubectl"
    }
  }
# eks node
  resource "aws_eks_node_group" "node-grp" {
    cluster_name    = aws_eks_cluster.eks.name
    node_group_name = "pc-node-group"
    node_role_arn   = aws_iam_role.worker.arn
    subnet_ids      = [aws_subnet.Mysubnet01.id, aws_subnet.Mysubnet02.id]
    capacity_type   = "ON_DEMAND"
    disk_size       = 20
    instance_types  = ["t2.small"]

    remote_access {
      ec2_ssh_key               = "mumbai-kp"
      source_security_group_ids = [aws_security_group.allow_tls.id]
    }

    labels = {
      env = "dev"
    }

    scaling_config {
      desired_size = 2
      max_size     = 2
      min_size     = 1
    }

    update_config {
      max_unavailable = 1
    }

    depends_on = [
      aws_iam_role_policy_attachment.AmazonEKSWorkerNodePolicy,
      aws_iam_role_policy_attachment.AmazonEKS_CNI_Policy,
      aws_iam_role_policy_attachment.AmazonEC2ContainerRegistryReadOnly,
    ]
  }
# Use this command to check syntax validations

* terraform validate

After that, we can use another command which is

* terraform plan
Now use the command, this command will tell us what changes it is going to make in our AWS


After that we apply the chnages

* terraform apply --auto-approve

After that you see the resources

EC2 Instances

VPC

EKS CLUSTER

Now we have to execute some more commands also

# Launch Kubectl Server
Configure Kubectl

In the curl command, I am using Kubernetes 1.23 you can use any latest

* curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.23.17/2023-05-11/bin/linux/amd64/kubectl

* openssl sha1 -sha256 kubectl

* chmod +x ./kubectl

* mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH

* kubectl version --short --client
* aws configure

See the below command and replace it with the name of your EKS cluster and with the AWS region where the cluster is located.

* aws eks update-kubeconfig - name <your-cluster-name> - region <your-region>
* kubectl get nodes
