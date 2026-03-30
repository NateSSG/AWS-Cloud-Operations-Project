# 

## Phase 1: Victim Web Server Deployment (Intrusion Detection System Project)

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

# Phase 2: Cloud Governance and Monitoring (Intrusion Detection System)

## 1. Introduction & Goal
The goal of Phase 2 is to wrap our vulnerable Phase 1 architecture in a layer of Cloud Governance and Security Monitoring. We are implementing an automated log analysis pipeline that detects unauthorized access attempts (SSH brute-forcing) and alerts the administrator in real-time using AWS native services.

## 2. Architecture & Resources Created
* **Amazon CloudWatch Log Group:** Centralized storage for the EC2 instance's OS-level security logs (`/var/log/secure`).
* **Amazon CloudWatch Metric Filter:** A custom monitoring rule that scans incoming logs in real-time for the string `"Invalid user"`.
* **Amazon CloudWatch Alarm:** A threshold monitor (`SSH-BruteForce-Alert`) configured to trigger if 3 or more failed login attempts occur within a 60-second window.
* **Amazon SNS (Simple Notification Service):** A notification topic (`SecurityAlerts`) that emails the administrator when the CloudWatch Alarm enters the "In Alarm" state.
* **AWS Systems Manager (SSM):** Used to remotely configure and manage the CloudWatch Agent and OS packages on the EC2 instance without requiring manual SSH access.

---

## 3. Step-by-Step Implementation (AWS CLI)

All governance and monitoring infrastructure was deployed via the AWS CLI in CloudShell.

### Step 1: Create the Alerting System (SNS)
Created an SNS Topic and subscribed an administrator email address to receive security alerts.
```bash
TOPIC_ARN=$(aws sns create-topic --name SecurityAlerts --query 'TopicArn' --output text)

aws sns subscribe \
    --topic-arn $TOPIC_ARN \
    --protocol email \
    --notification-endpoint my-email@example.com
```
<img width="943" height="410" alt="image" src="https://github.com/user-attachments/assets/d38ce620-93fa-49d2-a634-341afd35af8f" />


<img width="848" height="115" alt="image" src="https://github.com/user-attachments/assets/988d7d0f-83a9-4720-8470-152e419ecaec" />



### Step 2: Establish Centralized Logging

Created a CloudWatch Log Group to catch the authentication logs.

```bash
aws logs create-log-group --log-group-name /var/log/secure
```

### Step 3: Create the intrustion Detection Metric

Configured a metric filter to count occurences of "Invalid user" (which indicates a failed SSH login attempt with a non-existent username).

```bash
aws logs put-metric-filter \
  --log-group-name /var/log/secure \
  --filter-name "FailedSSHFilter" \
  --filter-pattern '"Invalid user"' \
  --metric-transformations \
      metricName=FailedSSHLogins,metricNamespace=SecurityMetrics,metricValue=1,defaultValue=0
```
### Step 4: Create the Security Alarm

Set the threshold to trigger if the attacker fails 3 times within 1 minute.

```bash
aws cloudwatch put-metric-alarm \
    --alarm-name "SSH-BruteForce-Alert" \
    --alarm-description "Triggers on 3+ failed SSH logins in 1 min." \
    --metric-name FailedSSHLogins \
    --namespace SecurityMetrics \
    --statistic Sum \
    --period 60 \
    --threshold 3 \
    --comparison-operator GreaterThanOrEqualToThreshold \
    --evaluation-periods 1 \
    --alarm-actions $TOPIC_ARN \
    --treat-missing-data notBreaching
```

### Step 5: Configure and Start the CloudWatch Agent

Used AWS Systems Manager Parameter Store to define the agent's configuration, then sent a remote command to the EC2 instance to apply the configuration and restart the agent.

```bash
# 1. Store the Configuration
aws ssm put-parameter \
    --name "AmazonCloudWatch-IDS-Config" \
    --type "String" \
    --value '{"logs": {"logs_collected": {"files": {"collect_list": [{"file_path": "/var/log/secure","log_group_name": "/var/log/secure","log_stream_name": "{instance_id}"}]}}}}' \
    --overwrite

# 2. Deploy to the EC2 Instance
aws ssm send-command \
    --targets "Key=tag:Name,Values=VictimServer" \
    --document-name "AmazonCloudWatch-ManageAgent" \
    --parameters '{"action":["configure"],"mode":["ec2"],"optionalConfigurationSource":["ssm"],"optionalConfigurationLocation":["AmazonCloudWatch-IDS-Config"],"optionalRestart":["yes"]}'
```

## Troubleshooting & Learnings

### Issue: Logs not appearing in CloudWatch

<img width="1274" height="810" alt="image" src="https://github.com/user-attachments/assets/336f6997-9159-40c7-bc73-be6207b5c08c" />

<img width="1272" height="856" alt="image" src="https://github.com/user-attachments/assets/b7455d96-24d7-49da-bb20-54071f009e71" />

- What went wrong? After deploying the CloudWatch Agent, the SSH-BruteForce-Alert alarm remained in the INSUFFICIENT_DATA state.

- How did I diagnose the issue? I opened the CloudWatch Log Management console and inspected the "/var/log/secure log" group. The group existed, but there were zero log events populated inside it, despite the EC2 instance running and the agent being active.

- What steps were taken to fix it? Upon researching the AMI used in Phase 1 (Amazon Linux 2023), I discovered that AWS removed the "rsyslog" service by default in this OS version. Therefore, the OS was not actually writing authentication attemps to the /var/log/secure file for the agent to read.

- The Fix. I used AWS Systems Manager (SSM) RunShellScript to remotely install, enable and start the missing rsyslog service, and restarted the SSH daemon, entirely bypassing the need to SSH into the machine manually.

```bash
aws ssm send-command \
    --targets "Key=tag:Name,Values=VictimServer" \
    --document-name "AWS-RunShellScript" \
    --parameters 'commands=["yum install -y rsyslog","systemctl start rsyslog","systemctl enable rsyslog","systemctl restart sshd"]'
```
## Validation & Testing

To verify the Intrustion Detection System was fully operational, I simulated and external brute-force attack from my laptop.

<img width="1191" height="527" alt="bruteforce" src="https://github.com/user-attachments/assets/0a68515f-a3af-42bc-8750-bf69acbfb907" />

- From my laptop (simulating the attacker), i repeatedly attempted to SSH into the EC2 instance using an invalid username  (ssh hacker@<PUBLIC_IP>) and random passwords.

- I executed this 5 times in rapid succession to deliberately breach the alarm threshold of "3 attempts per minute".

- I monitored the CloudWatch dashboard, observing the logs populate in real-time, and watched the alarm state shift from "OK" to "In alarm".

- Shortly after, I received the automated SNS email alert detailing the breach.

<img width="951" height="724" alt="image" src="https://github.com/user-attachments/assets/d3ab4cb3-fa99-44ac-b4a4-82642eaab3f9" />

<img width="2551" height="838" alt="image" src="https://github.com/user-attachments/assets/f8981953-d547-4828-8031-70f246341bd3" />

<img width="2255" height="378" alt="image" src="https://github.com/user-attachments/assets/66bcbcd1-8f31-4477-a0ef-e86aa44ec913" />

<img width="2559" height="635" alt="image" src="https://github.com/user-attachments/assets/8dee1fca-3939-424a-80b9-4bcad4552638" />

<img width="2559" height="797" alt="image" src="https://github.com/user-attachments/assets/fb7cf98d-b67d-416d-825c-d265cd713c53" />

