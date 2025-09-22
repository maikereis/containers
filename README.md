# Deploying and APP on AWS Container

This markdown demonstrates container deployment on AWS using Docker, CloudFormation, EC2.

## Prerequisites

- AWS CLI configured with appropriate permissions
- Basic familiarity with Docker and AWS services

## Architecture Overview

The CloudFormation template provisions:
- **VPC** with public subnet and internet gateway
- **IAM roles** with necessary permissions for container execution
- **Security group** with configurable access rules

## Quick Start

### 1. Deploy Infrastructure

```bash
aws cloudformation create-stack \
  --stack-name containers-stack \
  --template-body file://containers-stack.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

Monitor deployment status:
```bash
aws cloudformation describe-stacks \
  --stack-name containers-stack \
  --region us-east-1 \
  --query "Stacks[0].StackStatus"
```

### 2. Configure Environment

Set up authentication and environment variables:
```bash
# Generate EC2 key pair
aws ec2 create-key-pair --key-name container_keys \
  --query 'KeyMaterial' --output text > ~/.ssh/container_keys.pem
chmod 400 ~/.ssh/container_keys.pem

# Environment variables
export STACK_NAME="containers-stack"
export KEY_NAME="container_keys"
export INSTANCE_TYPE="t3.micro"
```

### 3. Launch EC2 Instance

Retrieve CloudFormation outputs:
```bash
SUBNET_ID=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='PublicSubnetId'].OutputValue" \
  --output text)

SECURITY_GROUP_ID=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='SecurityGroupId'].OutputValue" \
  --output text)

INSTANCE_PROFILE=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --query "Stacks[0].Outputs[?OutputKey=='InstanceProfileArn'].OutputValue" \
  --output text | cut -d'/' -f2)
```

Create user data for Docker installation:
```bash
USER_DATA=$(cat <<'EOF'
#!/bin/bash
yum update -y
yum install -y docker
systemctl start docker
systemctl enable docker
usermod -a -G docker ec2-user
EOF
)
```

Launch instance:
```bash
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id ami-0c02fb55956c7d316 \
  --count 1 \
  --instance-type $INSTANCE_TYPE \
  --key-name $KEY_NAME \
  --security-group-ids $SECURITY_GROUP_ID \
  --subnet-id $SUBNET_ID \
  --iam-instance-profile Name=$INSTANCE_PROFILE \
  --associate-public-ip-address \
  --user-data "$USER_DATA" \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=ContainerExercise}]" \
  --query 'Instances[0].InstanceId' \
  --output text)

aws ec2 wait instance-running --instance-ids $INSTANCE_ID
```

### 4. Configure Network Access

Get instance IP and configure SSH access:
```bash
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

# Allow SSH from your IP
MY_IP=$(curl -s http://checkip.amazonaws.com)
aws ec2 authorize-security-group-ingress \
  --group-id $SECURITY_GROUP_ID \
  --protocol tcp \
  --port 22 \
  --cidr ${MY_IP}/32

# Allow application access on port 8080
aws ec2 authorize-security-group-ingress \
  --group-id $SECURITY_GROUP_ID \
  --protocol tcp \
  --port 8080 \
  --cidr 0.0.0.0/0

echo "SSH: ssh -i ~/.ssh/$KEY_NAME.pem ec2-user@$PUBLIC_IP"
echo "App: http://$PUBLIC_IP:8080"
```

## Container Operations

### Basic Container Deployment

Connect to your instance and download the application:
```bash
ssh -i ~/.ssh/container_keys.pem ec2-user@$PUBLIC_IP

wget https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/DEV-AWS-MO-ContainersRedux/downloads/containers-src.zip
unzip containers-src.zip
cd first-container
```

Build and run the container:
```bash
docker build -t first-container .
docker run -d -p 8080:8080 --name webapp first-container
```

### Container Management

View logs and access container shell:
```bash
docker logs webapp
docker exec -it webapp /bin/sh
```

Stop and remove container:
```bash
docker stop webapp
docker rm webapp
```

### Advanced Configuration

Run container with environment variables and volume mounting:
```bash
# Prepare input file
printf "cinco\ncuatro\ntres\ndos\nuno" > ~/input.txt

# Run with custom configuration
docker run -d -p 8080:8080 \
  -e MESSAGE_COLOR=#0000ff \
  -v ~/input.txt:/app/input.txt \
  --name webapp \
  first-container
```

## Troubleshooting

### CloudFormation Issues
Check stack events for failed resources:
```bash
aws cloudformation describe-stack-events \
  --stack-name containers-stack \
  --region us-east-1 \
  --query "StackEvents[?ResourceStatus=='CREATE_FAILED'].[LogicalResourceId, ResourceStatusReason]"
```

### Container Issues
- Verify Docker service status: `sudo systemctl status docker`
- Check container logs: `docker logs webapp`
- Ensure security group allows required ports

## Cleanup

Remove all resources when finished:
```bash
# Terminate EC2 instance
aws ec2 terminate-instances --instance-ids $INSTANCE_ID

# Delete CloudFormation stack
aws cloudformation delete-stack --stack-name containers-stack --region us-east-1
```