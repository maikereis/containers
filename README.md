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

![](diagram.png)

## Quick Start

### 1. Configure Environment

Set up authentication and environment variables:
```bash
# Generate EC2 key pair
KEY_NAME="container-keys-$(date +%Y%m%d)"
aws ec2 create-key-pair --key-name $KEY_NAME \
  --query 'KeyMaterial' --output text > ~/.ssh/${KEY_NAME}.pem
chmod 600 ~/.ssh/${KEY_NAME}.pem

# Environment variables
export STACK_NAME="containers-stack"
export KEY_NAME="container_keys"
export INSTANCE_TYPE="t3.micro"
export AWS_REGION="us-east-1"
```


### 2. Deploy Infrastructure

```bash
aws cloudformation create-stack \
  --stack-name containers-stack \
  --template-body file://containers-stack.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region $AWS_REGION
```

Monitor deployment status:
```bash
aws cloudformation describe-stacks \
  --stack-name containers-stack \
  --region $AWS_REGION \
  --query "Stacks[0].StackStatus"
```

### 3. Launch EC2 Instance

Retrieve CloudFormation outputs:
```bash
SUBNET_ID=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --region $AWS_REGION \
  --query "Stacks[0].Outputs[?OutputKey=='PublicSubnetId'].OutputValue" \
  --output text)

SECURITY_GROUP_ID=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --region $AWS_REGION \
  --query "Stacks[0].Outputs[?OutputKey=='SecurityGroupId'].OutputValue" \
  --output text)

INSTANCE_PROFILE=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --region $AWS_REGION \
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
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text \
  --region $AWS_REGION)

INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --count 1 \
  --instance-type $INSTANCE_TYPE \
  --key-name $KEY_NAME \
  --security-group-ids $SECURITY_GROUP_ID \
  --subnet-id $SUBNET_ID \
  --iam-instance-profile Name=$INSTANCE_PROFILE \
  --associate-public-ip-address \
  --user-data "$USER_DATA" \
  --metadata-options "HttpTokens=required,HttpPutResponseHopLimit=2,HttpEndpoint=enabled" \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=ContainerExercise},{Key=Environment,Value=Development}]" \
  --query 'Instances[0].InstanceId' \
  --output text \
  --region $AWS_REGION)

aws ec2 wait instance-running --instance-ids $INSTANCE_ID --region $AWS_REGION
```

### 4. Configure Network Access

Get instance IP and configure SSH access:
```bash
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --region $AWS_REGION \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

# Allow SSH from your IP
MY_IP=$(curl -s http://checkip.amazonaws.com)
echo "Restricting SSH access to IP: $MY_IP"

aws ec2 authorize-security-group-ingress \
  --group-id $SECURITY_GROUP_ID \
  --protocol tcp \
  --port 22 \
  --cidr ${MY_IP}/32 \
  --region $AWS_REGION

echo "SSH: ssh -i ~/.ssh/$KEY_NAME.pem ec2-user@$PUBLIC_IP"
echo "Instance ID: $INSTANCE_ID"

# Allow application access on port 8080
aws ec2 authorize-security-group-ingress \
  --group-id $SECURITY_GROUP_ID \
  --protocol tcp \
  --port 8080 \
  --cidr ${MY_IP}/32 \
  --region $AWS_REGION 

echo "App: http://$PUBLIC_IP:8080 (accessible from your IP only)"
```

## Container Operations

### Basic Container Deployment

Connect to your instance and download the application:
```bash

ssh -i ~/.ssh/$KEY_NAME.pem ec2-user@$PUBLIC_IP

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
docker stop webapp 2>/dev/null || true
docker rm webapp 2>/dev/null || true
```

### Advanced Configuration

Run container with environment variables and volume mounting:
```bash
# Prepare input file
printf "cinco\ncuatro\ntres\ndos\nuno" > ~/input.txt

# Run with custom configuration
docker run -d \
  -p 8080:8080 \
  -e MESSAGE_COLOR=#0000ff \
  -v ~/input.txt:/app/input.txt:ro \
  --name webapp \
  --restart=unless-stopped \
  --read-only \
  --user 1001:1001 \
  --security-opt=no-new-privileges:true \
  --memory=256m \
  --cpus=0.5 \
  first-container

docker stop webapp 2>/dev/null || true
docker rm webapp 2>/dev/null || true
```

## Troubleshooting

### CloudFormation Issues
Check stack events for failed resources:
```bash
aws cloudformation describe-stack-events \
  --stack-name containers-stack \
  --region $AWS_REGION \
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
aws ec2 terminate-instances --instance-ids $INSTANCE_ID --region $AWS_REGION
# wait
aws ec2 wait instance-terminated --instance-ids $INSTANCE_ID --region $AWS_REGION
 
# Delet keys
aws ec2 delete-key-pair --key-name $KEY_NAME --region $AWS_REGION
rm ~/.ssh/${KEY_NAME}.pem

# Delete CloudFormation stack
aws cloudformation delete-stack --stack-name containers-stack --region $AWS_REGION
# wait
aws cloudformation wait stack-delete-complete --stack-name containers-stack --region $AWS_REGION
```