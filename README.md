# Apache-Microservice-Demo

--

```markdown
# 🚀 Apache Microservice Demo on AWS (CLI-Based)

This project demonstrates how to deploy a simple microservice architecture using Apache web servers on EC2 instances, an Application Load Balancer (ALB) with path-based routing, and Auto Scaling (only for the Shop microservice) — entirely via AWS CLI.

---

## ✅ Environment Setup

Update these values as needed:

```bash
export AWS_REGION="us-east-1"
export AMI_ID="ami-0c7217cdde317cfec"
export INSTANCE_TYPE="t2.micro"
export KEY_NAME="ec2-devops-key"
export VPC_ID="vpc-03701e181332d26eb"
export SUBNETS="subnet-062bafb72ff1b9c71,subnet-00f1308ab05d4d97a"
export SECURITY_GROUP_NAME="apache-microservice-sg"
```

---

## 🔐 Step 1: Create Security Group (HTTP on port 80 from ALB only)

```bash
aws ec2 create-security-group \
  --group-name $SECURITY_GROUP_NAME \
  --description "Security group for Apache microservice demo" \
  --vpc-id $VPC_ID

SG_ID=$(aws ec2 describe-security-groups \
  --filters Name=group-name,Values=$SECURITY_GROUP_NAME \
  --query "SecurityGroups[0].GroupId" --output text)

# Allow HTTP from anywhere (for ALB testing) — for production, allow only from ALB SG
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

---

## 🖥️ Step 2: Create EC2 Instances (Shop and Checkout)

### Create user data scripts:

```bash
mkdir scripts

cat <<'EOF' > scripts/shop-user-data.sh
#!/bin/bash
apt update -y
apt install apache2 -y
systemctl enable apache2
mkdir -p /var/www/html/shop/
echo "<h1>This is the shop instance.<br>Welcome to our SHOP.<br>Please, Click on shopping link below to make an order.<br>You will be directed to a checkout page.<br>Hope you found all you need.</h1>" > /var/www/html/shop/index.html
systemctl start apache2
EOF

cat <<'EOF' > scripts/checkout-user-data.sh
#!/bin/bash
apt update -y
apt install apache2 -y
systemctl enable apache2
mkdir -p /var/www/html/checkout/
echo "<h1>This is the checkout instance.<br>Please, go to checkout.<br>Thank you for being a valuable customer.<br>See You Next Time....</h1>" > /var/www/html/checkout/index.html
systemctl start apache2
EOF
```

### Launch EC2 Instances:

```bash
# Shop instance
aws ec2 run-instances \
  --image-id $AMI_ID \
  --count 1 \
  --instance-type $INSTANCE_TYPE \
  --key-name $KEY_NAME \
  --security-group-ids $SG_ID \
  --subnet-id subnet-062bafb72ff1b9c71 \
  --user-data file://scripts/shop-user-data.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Shop-Service}]'

# Checkout instance
aws ec2 run-instances \
  --image-id $AMI_ID \
  --count 1 \
  --instance-type $INSTANCE_TYPE \
  --key-name $KEY_NAME \
  --security-group-ids $SG_ID \
  --subnet-id subnet-062bafb72ff1b9c71 \
  --user-data file://scripts/checkout-user-data.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Checkout-Service}]'
```

---

## 🎯 Step 3: Create Target Groups

```bash
# Shop target group
aws elbv2 create-target-group \
  --name shop-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id $VPC_ID \
  --target-type instance \
  --health-check-path "/shop/index.html"

# Checkout target group
aws elbv2 create-target-group \
  --name checkout-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id $VPC_ID \
  --target-type instance \
  --health-check-path "/checkout/index.html"
```

---

## 🌐 Step 4: Create Application Load Balancer

```bash
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name apache-microservice-alb \
  --subnets subnet-062bafb72ff1b9c71 subnet-00f1308ab05d4d97a \
  --security-groups $SG_ID \
  --scheme internet-facing \
  --type application \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text)
```

---

## 🔁 Step 5: Create Listener with Routing Rules

### Get Target Group ARNs:

```bash
SHOP_TG_ARN=$(aws elbv2 describe-target-groups --names shop-tg --query 'TargetGroups[0].TargetGroupArn' --output text)
CHECKOUT_TG_ARN=$(aws elbv2 describe-target-groups --names checkout-tg --query 'TargetGroups[0].TargetGroupArn' --output text)
```

### Create Listener:

```bash
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=fixed-response,FixedResponseConfig='{StatusCode=200,ContentType=text/plain,MessageBody="Welcome to Apache Microservice Demo!"}'
```

### Add Path-Based Rules:

```bash
LISTENER_ARN=$(aws elbv2 describe-listeners \
  --load-balancer-arn $ALB_ARN \
  --query 'Listeners[0].ListenerArn' \
  --output text)

# /shop*
aws elbv2 create-rule \
  --listener-arn $LISTENER_ARN \
  --priority 10 \
  --conditions Field=path-pattern,Values='/shop*' \
  --actions Type=forward,TargetGroupArn=$SHOP_TG_ARN

# /checkout*
aws elbv2 create-rule \
  --listener-arn $LISTENER_ARN \
  --priority 20 \
  --conditions Field=path-pattern,Values='/checkout*' \
  --actions Type=forward,TargetGroupArn=$CHECKOUT_TG_ARN
```

---

## 📈 Step 6: Enable Auto Scaling for Shop

### Encode Shop User Data:

```bash
USER_DATA_BASE64=$(base64 -w0 scripts/shop-user-data.sh)
```

### Create Launch Template:

```bash
aws ec2 create-launch-template \
  --launch-template-name shop-launch-template \
  --version-description "Shop Auto Scaling" \
  --launch-template-data "{
    \"ImageId\": \"$AMI_ID\",
    \"InstanceType\": \"$INSTANCE_TYPE\",
    \"KeyName\": \"$KEY_NAME\",
    \"SecurityGroupIds\": [\"$SG_ID\"],
    \"UserData\": \"$USER_DATA_BASE64\"
  }"
```

### Create Auto Scaling Group:

```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name shop-asg \
  --launch-template LaunchTemplateName=shop-launch-template,Version=1 \
  --min-size 1 \
  --max-size 3 \
  --desired-capacity 1 \
  --vpc-zone-identifier $SUBNETS \
  --target-group-arns $SHOP_TG_ARN
```

---

## ✅ Done!

You now have:

- EC2 instances for Shop and Checkout
- Path-based routing using an Application Load Balancer
- Security Group protecting direct access to EC2
- Auto Scaling enabled for Shop only

---

## 📌 Next Steps

- 🔐 Modify the security group to accept traffic **only from the ALB**
- 📊 Add CloudWatch alarms and scaling policies (CPU-based)
- 💡 Convert to Terraform for infrastructure-as-code

---
