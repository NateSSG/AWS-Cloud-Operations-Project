# Phase 1: Victim Web Server Deployment (Intrusion Detection System Project)

## 1. Introduction & Goal
The goal of Phase 1 is to deploy the baseline "victim" infrastructure for an Automated Intrusion Detection System (IDS). We are setting up a simple, intentionally accessible web server using Amazon EC2. This server will act as the target for simulated attacks (like SSH brute-forcing) in Phase 2, allowing us to build automated detection and response mechanisms around it.

## 2. Architecture & Resources Created
* **Amazon EC2 (`t2.micro`):** Runs Amazon Linux 2023 and an Apache Web Server. 
* **Security Group (`VictimWebServerSG`):** Acts as the instance firewall. It allows incoming HTTP (Port 80) traffic so the website is visible, and SSH (Port 22) traffic from anywhere (`0.0.0.0/0`) to allow us to simulate SSH attacks later.
* **IAM Instance Profile (`LabInstanceProfile`):** Attached to the EC2 instance to grant it permission to communicate with CloudWatch in Phase 2.
* **Key Pair (`vockey`):** The default AWS Academy Learner Lab key used for SSH access.




---

## 3. Step-by-Step Implementation (AWS CLI)

The entire Phase 1 infrastructure was deployed using Infrastructure as Code (IaC) via the AWS CLI within AWS CloudShell. 

### Step 1: Create the Security Group
Created a security group to act as the firewall for our web server.
```bash
aws ec2 create-security-group \
    --group-name VictimWebServerSG \
    --description "Allow HTTP and SSH for IDS Phase 1"
```

### Step 2: Configure Firewall Rules

Opened ports 80 (Web) and 22 (SSH) to the public. Port 22 is left wide open intentionally for the scope of this project to capture attack logs.

```bash
aws ec2 authorize-security-group-ingress \
    --group-name VictimWebServerSG \
    --protocol tcp --port 22 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
    --group-name VictimWebServerSG \
    --protocol tcp --port 80 --cidr 0.0.0.0/0
```
### Step 3: Prepare the Web Server Installation Script

Created a "user_data.sh" script to automatically install Apache and the CloudWatch Agent (Preparation for Phase 2) as soon as the server boots.

```bash
cat <<EOF > user_data.sh
#!/bin/bash
yum update -y
yum install -y httpd amazon-cloudwatch-agent
systemctl start httpd
systemctl enable httpd
echo "<html><body style='font-family: Arial; text-align: center; margin-top: 50px;'><h1>⚠️ Victim Web Server Running ⚠️</h1><p>Phase 1 Infrastructure is successfully deployed!</p></body></html>" > /var/www/html/index.html
EOF
```
### Step 4: Fetch the Latest Amazon Linux AMI

Dynamically queried AWS to find the most up to date Amazon Linux 2023 Image ID.

```bash
AMI_ID=$(aws ssm get-parameters --names /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64 --query 'Parameters[0].Value' --output text)
```

### Step 5: Launch the EC2 Instance

Deployed the server using the config above, attaching the Learner Lab mandated "vockey" key and "LabInstanceProfile".

```bash
aws ec2 run-instances \
    --image-id $AMI_ID \
    --count 1 \
    --instance-type t2.micro \
    --key-name vockey \
    --security-groups VictimWebServerSG \
    --iam-instance-profile Name="LabInstanceProfile" \
    --user-data file://user_data.sh \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=VictimServer}]'
```
### Step 6: Retrieve Public IP 

Fetched the assigned Public IP address to validate the deployment.

```bash
aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=VictimServer" "Name=instance-state-name,Values=running" \
    --query 'Reservations[*].Instances[*].PublicIpAddress' \
    --output text
```

## Validation & Testing

To verify that the outcome is working as expected:

- I copied the Public IP address generated from step 6.
- I pasted the IP address into a standard web browser over HTTP (e.g., http://3.93.167.231).
- The custom "Victim Web Server Running" HTML page loaded successfully, proving that the EC2 instance booted, the User       Data script executed successfully, and the Security Group is correctly allowing Port 80 traffic.

<img width="1275" height="974" alt="image" src="https://github.com/user-attachments/assets/b6f0ade2-17b3-44e1-864d-79d3ff44a4df" />

<img width="1161" height="909" alt="image" src="https://github.com/user-attachments/assets/15330f6d-3920-4d0f-845f-aa2a3de3bb31" />


<img width="943" height="410" alt="image" src="https://github.com/user-attachments/assets/d38ce620-93fa-49d2-a634-341afd35af8f" />


<img width="848" height="115" alt="image" src="https://github.com/user-attachments/assets/988d7d0f-83a9-4720-8470-152e419ecaec" />


