# Secure-AWS-Web-App
a three-tier AWS architecture with security best practices (VPC, IAM, CloudTrail, GuardDuty)

# PROJECT VARIABLES
$PROJECT="CloudSecProj"
$REGION="us-east-1"

# NETWORK SETTINGS
$VPC_CIDR="10.0.0.0/16"
$PUB_SUB1="10.0.1.0/24"
$PUB_SUB2="10.0.2.0/24"
$PRIV_SUB1="10.0.3.0/24"
$PRIV_SUB2="10.0.4.0/24"

# Create VPC
$VPC_ID = (aws ec2 create-vpc --cidr-block $VPC_CIDR --query "Vpc.VpcId" --output text)
aws ec2 create-tags --resources $VPC_ID --tags Key=Name,Value="$PROJECT-VPC"
echo "✅ Created VPC: $VPC_ID"

# CREATE INTERNET GATEWAY
$IGW_ID = (aws ec2 create-internet-gateway --query "InternetGateway.InternetGatewayId" --output text)
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID
echo "✅ Created IGW: $IGW_ID"

# Public Table Route
$RTB_PUBLIC = (aws ec2 create-route-table --vpc-id $VPC_ID --query "RouteTable.RouteTableId" --output text)
aws ec2 create-route --route-table-id $RTB_PUBLIC --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $RTB_PUBLIC --subnet-id $PUB1_ID
aws ec2 associate-route-table --route-table-id $RTB_PUBLIC --subnet-id $PUB2_ID
echo "✅ Public Route Table: $RTB_PUBLIC"

# AWS Security Groups (PUBLIC)
$WEB_SG = (aws ec2 create-security-group --group-name WebSG --description "Allow HTTP/HTTPS" --vpc-id $VPC_ID --query "GroupId" --output text)
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 22 --cidr 0.0.0.0/0
echo "✅ Web SG: $WEB_SG"

# App Security Group (Private) 
$APP_SG = (aws ec2 create-security-group --group-name AppSG --description "Allow Web Tier Only" --vpc-id $VPC_ID --query "GroupId" --output text)
aws ec2 authorize-security-group-ingress --group-id $APP_SG --protocol tcp --port 3306 --source-group $WEB_SG
echo "✅ App SG: $APP_SG"

# Launch EC2 Instances
aws ec2 create-key-pair --key-name $PROJECT-key --query "KeyMaterial" --output text > $HOME\$PROJECT-key.pem
echo "✅ Key pair created: $PROJECT-key.pem"

# Launch Web Server
$WEB_ID = (aws ec2 run-instances --image-id ami-0c02fb55956c7d316 --count 1 --instance-type t2.micro --key-name $PROJECT-key --security-group-ids $WEB_SG --subnet-id $PUB1_ID --associate-public-ip-address --query "Instances[0].InstanceId" --output text)
echo "✅ Web EC2: $WEB_ID"


# Launch App Server
$APP_ID = (aws ec2 run-instances --image-id ami-0c02fb55956c7d316 --count 1 --instance-type t2.micro --key-name $PROJECT-key --security-group-ids $APP_SG --subnet-id $PRIV1_ID --query "Instances[0].InstanceId" --output text)
echo "✅ App EC2: $APP_ID"

# Enable AWS Security Services

# Guard Duty
aws guardduty create-detector --enable
echo "✅ GuardDuty enabled."

# CloudTrail
$BUCKET_NAME="cloudsec-trail-$(Get-Random)"
aws s3 mb s3://$BUCKET_NAME
aws cloudtrail create-trail --name $PROJECT-Trail --s3-bucket-name $BUCKET_NAME
aws cloudtrail start-logging --name $PROJECT-Trail
echo "✅ CloudTrail enabled. Logs going to bucket: $BUCKET_NAME"
