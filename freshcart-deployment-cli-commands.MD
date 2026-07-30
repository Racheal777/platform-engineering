# FreshCart Two-Tier Architecture — CLI Commands

**Provider:** AWS
**Region:** eu-north-1 (Stockholm)

---

## 1. VPC

```bash
# Create the VPC (the fenced private network everything lives inside)
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=freshcart-vpc}]' \
  --region eu-north-1
```

## 2. Subnets

```bash
# Public subnet in zone A
aws ec2 create-subnet \
  --vpc-id vpc-01e7dfe4dd68ccf5a \
  --cidr-block 10.0.1.0/24 \
  --availability-zone eu-north-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=freshcart-public-1a}]' \
  --region eu-north-1

# Private subnet in zone A
aws ec2 create-subnet \
  --vpc-id vpc-01e7dfe4dd68ccf5a \
  --cidr-block 10.0.2.0/24 \
  --availability-zone eu-north-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=freshcart-private-1a}]' \
  --region eu-north-1

# Public subnet in zone B
aws ec2 create-subnet \
  --vpc-id vpc-01e7dfe4dd68ccf5a \
  --cidr-block 10.0.3.0/24 \
  --availability-zone eu-north-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=freshcart-public-1b}]' \
  --region eu-north-1

# Private subnet in zone B
aws ec2 create-subnet \
  --vpc-id vpc-01e7dfe4dd68ccf5a \
  --cidr-block 10.0.4.0/24 \
  --availability-zone eu-north-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=freshcart-private-1b}]' \
  --region eu-north-1
```

## 3. Internet Gateway

```bash
# Create the single exit/entry point connecting VPC to the internet
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=freshcart-igw}]' \
  --region eu-north-1

# Attach it to the VPC
aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-0648775833b51077b \
  --vpc-id vpc-01e7dfe4dd68ccf5a \
  --region eu-north-1
```

## 4. Public Route Table

```bash
# Create route table for public subnets
aws ec2 create-route-table \
  --vpc-id vpc-01e7dfe4dd68ccf5a \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=freshcart-public-rt}]' \
  --region eu-north-1

# Add route to Internet Gateway (this is what makes a subnet "public")
aws ec2 create-route \
  --route-table-id rtb-0d938dbb102a4a816 \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-0648775833b51077b \
  --region eu-north-1

# Associate with both public subnets
aws ec2 associate-route-table \
  --route-table-id rtb-0d938dbb102a4a816 \
  --subnet-id subnet-02d6a71d80ae5e670 \
  --region eu-north-1

aws ec2 associate-route-table \
  --route-table-id rtb-0d938dbb102a4a816 \
  --subnet-id subnet-0b18f6d6ed8bc40f1 \
  --region eu-north-1
```

## 5. NAT Gateways

```bash
# Allocate Elastic IPs (permanent public IPs for the NAT Gateways)
aws ec2 allocate-address \
  --domain vpc \
  --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=freshcart-nat-eip-1a}]' \
  --region eu-north-1

aws ec2 allocate-address \
  --domain vpc \
  --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=freshcart-nat-eip-1b}]' \
  --region eu-north-1

# Create NAT Gateway in public subnet zone A (outbound internet for private backends)
aws ec2 create-nat-gateway \
  --subnet-id subnet-02d6a71d80ae5e670 \
  --allocation-id eipalloc-041f896db4a002882 \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=freshcart-nat-1a}]' \
  --region eu-north-1

# Create NAT Gateway in public subnet zone B
aws ec2 create-nat-gateway \
  --subnet-id subnet-0b18f6d6ed8bc40f1 \
  --allocation-id eipalloc-0cad42fe176c8467c \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=freshcart-nat-1b}]' \
  --region eu-north-1
```

## 6. Private Route Tables

```bash
# Create private route table for zone A
aws ec2 create-route-table \
  --vpc-id vpc-01e7dfe4dd68ccf5a \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=freshcart-private-rt-1a}]' \
  --region eu-north-1

# Create private route table for zone B
aws ec2 create-route-table \
  --vpc-id vpc-01e7dfe4dd68ccf5a \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=freshcart-private-rt-1b}]' \
  --region eu-north-1

# Route outbound traffic through NAT Gateway (zone A)
aws ec2 create-route \
  --route-table-id rtb-0f317ad92374aa55c \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-083087a2c36174ee6 \
  --region eu-north-1

# Route outbound traffic through NAT Gateway (zone B)
aws ec2 create-route \
  --route-table-id rtb-000d3b6324d94f6f7 \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-049e06fd7d25ab98a \
  --region eu-north-1

# Associate with private subnets
aws ec2 associate-route-table \
  --route-table-id rtb-0f317ad92374aa55c \
  --subnet-id subnet-0e65431d6ed90a52b \
  --region eu-north-1

aws ec2 associate-route-table \
  --route-table-id rtb-000d3b6324d94f6f7 \
  --subnet-id subnet-072e7614781f4231b \
  --region eu-north-1
```

## 7. Security Groups

```bash
# ALB security group (accepts traffic from the internet)
aws ec2 create-security-group \
  --group-name freshcart-alb-sg \
  --description "Allow HTTPS from internet to ALB" \
  --vpc-id vpc-01e7dfe4dd68ccf5a \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=freshcart-alb-sg}]' \
  --region eu-north-1

# Allow HTTPS from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id sg-02e683ae3aa6955fa \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0 \
  --region eu-north-1

# Allow HTTP from anywhere (for initial testing)
aws ec2 authorize-security-group-ingress \
  --group-id sg-02e683ae3aa6955fa \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0 \
  --region eu-north-1

# Backend security group (only accepts traffic from ALB)
aws ec2 create-security-group \
  --group-name freshcart-backend-sg \
  --description "Allow traffic from ALB only on port 3000" \
  --vpc-id vpc-01e7dfe4dd68ccf5a \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=freshcart-backend-sg}]' \
  --region eu-north-1

# Allow port 3000 from ALB's security group only
aws ec2 authorize-security-group-ingress \
  --group-id sg-03985da3ee6f2cee3 \
  --protocol tcp \
  --port 3000 \
  --source-group sg-02e683ae3aa6955fa \
  --region eu-north-1

# Allow SSH (for setup only, lock down in production)
aws ec2 authorize-security-group-ingress \
  --group-id sg-03985da3ee6f2cee3 \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0 \
  --region eu-north-1
```

## 8. EC2 Backend Instance

```bash
# Create key pair for SSH
aws ec2 create-key-pair \
  --key-name freshcart-key \
  --query 'KeyMaterial' \
  --output text \
  --region eu-north-1 > freshcart-key.pem

chmod 400 freshcart-key.pem

# Launch backend in private subnet (no public IP)
aws ec2 run-instances \
  --image-id ami-0cda11afd45b74b89 \
  --instance-type t3.micro \
  --key-name freshcart-key \
  --security-group-ids sg-03985da3ee6f2cee3 \
  --subnet-id subnet-0e65431d6ed90a52b \
  --no-associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=freshcart-backend-1a}]' \
  --region eu-north-1
```

## 9. Session Manager (IAM Role for SSM access)

```bash
# Create IAM role
aws iam create-role \
  --role-name freshcart-backend-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

# Attach SSM policy
aws iam attach-role-policy \
  --role-name freshcart-backend-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Create and attach instance profile
aws iam create-instance-profile \
  --instance-profile-name freshcart-backend-profile

aws iam add-role-to-instance-profile \
  --instance-profile-name freshcart-backend-profile \
  --role-name freshcart-backend-role

aws ec2 associate-iam-instance-profile \
  --iam-instance-profile Name=freshcart-backend-profile \
  --instance-id i-057bafc7a220f488d \
  --region eu-north-1

# Connect via Session Manager
aws ssm start-session \
  --target i-057bafc7a220f488d \
  --region eu-north-1
```

## 10. Application Load Balancer

```bash
# Create ALB spanning both public subnets
aws elbv2 create-load-balancer \
  --name freshcart-alb \
  --subnets subnet-02d6a71d80ae5e670 subnet-0b18f6d6ed8bc40f1 \
  --security-groups sg-02e683ae3aa6955fa \
  --scheme internet-facing \
  --type application \
  --region eu-north-1

# Create target group (where ALB forwards traffic)
aws elbv2 create-target-group \
  --name freshcart-targets \
  --protocol HTTP \
  --port 3000 \
  --vpc-id vpc-01e7dfe4dd68ccf5a \
  --health-check-path /health \
  --health-check-protocol HTTP \
  --target-type instance \
  --region eu-north-1

# Register backend instance with target group
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:eu-north-1:055505191746:targetgroup/freshcart-targets/603790cc2e746920 \
  --targets Id=i-057bafc7a220f488d \
  --region eu-north-1

# Create HTTP listener (port 80)
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:eu-north-1:055505191746:loadbalancer/app/freshcart-alb/8d94f239e8c11a98 \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:eu-north-1:055505191746:targetgroup/freshcart-targets/603790cc2e746920 \
  --region eu-north-1

# Create HTTPS listener (port 443 with certificate)
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:eu-north-1:055505191746:loadbalancer/app/freshcart-alb/8d94f239e8c11a98 \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:eu-north-1:055505191746:certificate/7908245c-fdd3-4bef-86af-86f469b5bb23 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:eu-north-1:055505191746:targetgroup/freshcart-targets/603790cc2e746920 \
  --region eu-north-1
```

## 11. HTTPS Certificate

```bash
# Request certificate from AWS Certificate Manager
aws acm request-certificate \
  --domain-name rachealplatformengineer.cloud-ip.cc \
  --validation-method DNS \
  --region eu-north-1

# Check validation status
aws acm describe-certificate \
  --certificate-arn arn:aws:acm:eu-north-1:055505191746:certificate/7908245c-fdd3-4bef-86af-86f469b5bb23 \
  --query 'Certificate.Status' \
  --output text \
  --region eu-north-1
```

## 12. S3 Static Site

```bash
# Create bucket
aws s3api create-bucket \
  --bucket freshcart-static-site-rk \
  --region eu-north-1 \
  --create-bucket-configuration LocationConstraint=eu-north-1

# Enable static website hosting
aws s3api put-bucket-website \
  --bucket freshcart-static-site-rk \
  --website-configuration '{"IndexDocument":{"Suffix":"index.html"},"ErrorDocument":{"Key":"error.html"}}'

# Allow public access
aws s3api put-public-access-block \
  --bucket freshcart-static-site-rk \
  --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

# Add public read policy
aws s3api put-bucket-policy \
  --bucket freshcart-static-site-rk \
  --policy '{"Version":"2012-10-17","Statement":[{"Sid":"PublicRead","Effect":"Allow","Principal":"*","Action":"s3:GetObject","Resource":"arn:aws:s3:::freshcart-static-site-rk/*"}]}'

# Upload files
aws s3 sync . s3://freshcart-static-site-rk/ --region eu-north-1

# Add lifecycle rule (30 days → Standard-IA, 90 days → delete)
aws s3api put-bucket-lifecycle-configuration \
  --bucket freshcart-static-site-rk \
  --lifecycle-configuration '{"Rules":[{"ID":"freshcart-lifecycle","Status":"Enabled","Filter":{"Prefix":""},"Transitions":[{"Days":30,"StorageClass":"STANDARD_IA"}],"Expiration":{"Days":90}}]}'
```

## 13. Backend Setup (run inside Session Manager)

```bash
sudo su - ubuntu
cd ~
git clone https://github.com/Racheal777/api-test-platform-engineering.git
cd api-test-platform-engineering
npm install
nohup node index.js > app.log 2>&1 &
curl http://localhost:3000/health
```

---

## Resource IDs Reference

| Resource | ID |
|---|---|
| VPC | vpc-01e7dfe4dd68ccf5a |
| Public subnet 1a | subnet-02d6a71d80ae5e670 |
| Private subnet 1a | subnet-0e65431d6ed90a52b |
| Public subnet 1b | subnet-0b18f6d6ed8bc40f1 |
| Private subnet 1b | subnet-072e7614781f4231b |
| Internet Gateway | igw-0648775833b51077b |
| Public route table | rtb-0d938dbb102a4a816 |
| Private route table 1a | rtb-0f317ad92374aa55c |
| Private route table 1b | rtb-000d3b6324d94f6f7 |
| NAT Gateway 1a | nat-083087a2c36174ee6 |
| NAT Gateway 1b | nat-049e06fd7d25ab98a |
| EIP 1a | eipalloc-041f896db4a002882 |
| EIP 1b | eipalloc-0cad42fe176c8467c |
| ALB SG | sg-02e683ae3aa6955fa |
| Backend SG | sg-03985da3ee6f2cee3 |
| Backend instance | i-057bafc7a220f488d |
| ALB | freshcart-alb-1513544613.eu-north-1.elb.amazonaws.com |
| Certificate | arn:aws:acm:eu-north-1:055505191746:certificate/7908245c-fdd3-4bef-86af-86f469b5bb23 |
| S3 static site | freshcart-static-site-rk.s3-website.eu-north-1.amazonaws.com |
| Domain | www.rachealplatformengineer.cloud-ip.cc |