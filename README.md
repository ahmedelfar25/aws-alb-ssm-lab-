# AWS ALB + EC2 + Systems Manager Lab

This repository contains an AWS lab project that builds a highly available web tier using **CloudFormation** and manages the EC2 instances using **AWS Systems Manager (SSM)**.

## Architecture Overview

- VPC 'VPC' with CIDR `10.0.0.0/16` in `us-east-1`.
- 3 public subnets in 3 different Availability Zones.
- PublicSubnet1 in use1-az1 (us-east-1a)
- PublicSubnet2 in use1-az1 (us-east-1b)
- PublicSubnet3 in use1-az1 (us-east-1c)
- 3 EC2 instances (Amazon Linux 2023) acting as web servers:
  - `webserver1` in PublicSubnet1
  - `webserver2` in PublicSubnet2
  - `webserver3` in PublicSubnet3
- Internet-facing Application Load Balancer (ALB) 'web-alb' in front of the 3 web servers.
- Instance target group 'webservers-tg' on port 80 registered with the 3 EC2 instances.
- Security group 'web-sg' allowing HTTP/HTTPS traffic and outbound internet access.
- IAM Role 'alb-ssm-ec2-role' and Instance Profile 'alb-ssm-ec2-instance-profile' attached to the EC2 instances with the managed policy `AmazonSSMManagedInstanceCore` to enable AWS Systems Manager features.

## Web Tier Behaviour

Each EC2 instance is launched in a different Availability Zone and bootstrapped via User Data:

- Updates system packages using `dnf`.
- Installs and starts Apache (`httpd`).
- Writes its own name to `/var/www/html/index.html`:
  - `webserver1` → page shows `webserver1`
  - `webserver2` → page shows `webserver2`
  - `webserver3` → page shows `webserver3`

## CloudFormation Template

The file `alb-ssm-lab.yaml` is the cloudformation template

## Deployment

The infrastructure is deployed using the AWS CLI and CloudFormation:

aws cloudformation deploy
--stack-name alb-ssm-lab
--template-file alb-ssm-lab.yaml
--capabilities CAPABILITY_NAMED_IAM

## Systems Manager Integration

All three EC2 instances are managed by AWS Systems Manager:

- The IAM Instance Profile attached to the instances includes `AmazonSSMManagedInstanceCore`, which allows the SSM Agent to register the instances as managed nodes.

### SSM Run Command

aws ssm send-command
--instance-ids i-06f43cb32db5c8de4 i-021284ea83d3dd32b i-094f1df992a298dc6
--document-name "AWS-RunShellScript"
--comment "Test SSM RunCommand on 3 webservers"
--parameters "commands=[""echo run-command-test >> /var/log/run-command.log""]"
--region us-east-1

This appends a line to `/var/log/run-command.log` on each instance.

### SSM State Manager

aws ssm create-association
--name "AWS-RunShellScript"
--targets "Key=tag:Name,Values=webserver1,webserver2,webserver3"
--parameters "commands=[""echo state-manager >> /var/log/state-manager.log""]"
--schedule-expression "rate(30 minutes)"
--association-name "webservers-state-manager"
--region us-east-1
