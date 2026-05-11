# Automated Intrusion Detection System | Nathaniel Ssendagire 11.05.2026

## Phase 1: Victim Web Server Deployment 

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

# Chapter 4: Architectural Enhancements & Cloud Governance

After successfully deploying the base Intrusion Detection System (Phase 1 & 2), I analyzed the architecture for potential improvements aligned with the AWS Well-Architected Framework. 

### Proposed Enhancements:
1. **Security & Governance (Automated Incident Response - SOAR):** Instead of just alerting the administrator via email when an attack occurs, the architecture can be enhanced to become "self-healing." By integrating an AWS Lambda function, the system can automatically parse logs, extract the attacker's IP, and inject a `DENY` rule into the VPC Network ACL.
2. **Cost Optimization:** To ensure the lab budget is preserved, I utilized the AWS Pricing Calculator to estimate costs and regularly checked **AWS Trusted Advisor** to identify and terminate idle EC2 instances when not actively testing.
3. **Resiliency (Automated Backups):** A potential future enhancement is utilizing Amazon Data Lifecycle Manager (DLM) to take automated daily EBS Snapshots of the web server. If an attacker successfully breached and corrupted the server before the auto-blocker fired, the server could be instantly restored.

For this project, I chose to implement **Enhancement 1: Automated Incident Response (The Auto-Blocker)**, as it provides the highest level of security automation and directly improves cloud governance.

---

# Chapter 5: Enhancement Implementation (The Auto-Blocker)

## 1. Architecture & Workflow
The goal of this enhancement is to transition from an Intrusion *Detection* System (IDS) to an Intrusion *Prevention* System (IPS). 
1. The **CloudWatch Alarm** (`SSH-BruteForce-Alert`) triggers after 3 failed logins in 60 seconds.
2. The Alarm sends a message to the **SNS Topic**.
3. The SNS Topic invokes a custom **AWS Lambda Function** (`AutoBlocker`) written in Python.
4. The Lambda function queries the CloudWatch Logs (`/var/log/secure`), uses Regular Expressions (Regex) to extract the attacker's IP address, and uses the `boto3` SDK to update the VPC **Network ACL**, instantly dropping all traffic from the attacker.

## 2. Infrastructure as Code (Deployment)
The Lambda function was packaged and deployed entirely via the AWS CLI in CloudShell. 

**The Python Script (ChatGPT) (`autoblocker.py`):**
```python
import boto3
import time
import re

def lambda_handler(event, context):
    logs_client = boto3.client('logs')
    ec2_client = boto3.client('ec2')

    print("Alert received! Scanning logs for attacker IP...")
    
    # 1. Query the last 5 minutes of logs
    end_time = int(time.time() * 1000)
    start_time = end_time - (5 * 60 * 1000)

    try:
        response = logs_client.filter_log_events(
            logGroupName='/var/log/secure',
            filterPattern='"Invalid user"',
            startTime=start_time,
            endTime=end_time
        )
    except Exception as e:
        print(f"Error reading logs: {e}")
        return

    # 2. Extract the IP address
    ips_to_block = set()
    for event in response.get('events', []):
        match = re.search(r'from\s+([0-9]+\.[0-9]+\.[0-9]+\.[0-9]+)', event['message'])
        if match:
            ips_to_block.add(match.group(1))

    if not ips_to_block:
        return {"status": "success", "message": "No IPs found."}

    # 3. Find the Default Network ACL
    vpcs = ec2_client.describe_vpcs(Filters=[{'Name': 'isDefault', 'Values': ['true']}])
    vpc_id = vpcs['Vpcs'][0]['VpcId']
    nacls = ec2_client.describe_network_acls(Filters=[{'Name': 'vpc-id', 'Values': [vpc_id]}])
    nacl_id = nacls['NetworkAcls'][0]['NetworkAclId']

    # 4. Inject the DENY rule
    rule_num = 50 
    for ip in ips_to_block:
        try:
            ec2_client.create_network_acl_entry(
                NetworkAclId=nacl_id,
                RuleNumber=rule_num,
                Protocol='-1', 
                RuleAction='deny',
                Egress=False, 
                CidrBlock=f"{ip}/32",
                PortRange={'From': 0, 'To': 65535}
            )
            print(f"SUCCESS: Blocked Hacker IP -> {ip}")
            rule_num += 1
        except Exception as e:
            print(f"Failed to block {ip}. AWS Error: {e}")

    return {"status": "success", "blocked_ips": list(ips_to_block)}

```
<img width="583" height="617" alt="autoblocker script code" src="https://github.com/user-attachments/assets/74825ffc-12c3-4344-8258-4b349ae16b52" />


CLI Deployment Commands: 

```bash
# Zip the code and create the Lambda using the Learner Lab Role
zip function.zip autoblocker.py
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws lambda create-function \
    --function-name AutoBlocker \
    --runtime python3.9 \
    --role arn:aws:iam::$ACCOUNT_ID:role/LabRole \
    --handler autoblocker.lambda_handler \
    --zip-file fileb://function.zip \
    --timeout 15

# Connect SNS to Lambda
TOPIC_ARN=$(aws sns list-topics --query "Topics[?contains(TopicArn, 'SecurityAlerts')].TopicArn" --output text)
LAMBDA_ARN=$(aws lambda get-function --function-name AutoBlocker --query 'Configuration.FunctionArn' --output text)

aws lambda add-permission --function-name AutoBlocker --statement-id sns-invoke --action "lambda:InvokeFunction" --principal sns.amazonaws.com --source-arn $TOPIC_ARN
aws sns subscribe --topic-arn $TOPIC_ARN --protocol lambda --notification-endpoint $LAMBDA_ARN
```
## 3. Troubleshooting & Learnings
### Issue: The Lambda executed, but the attacker was not blocked.

<img width="815" height="761" alt="alarm was triggered" src="https://github.com/user-attachments/assets/c1d781a4-a841-464e-b877-c9b8448278ad" />

<img width="985" height="396" alt="permission denied" src="https://github.com/user-attachments/assets/1e49801f-9dc8-413e-84f8-8fbc04090979" />

**What went wrong?** During the initial test, the attack triggered the alarm, but my attacking machine still received a "Permission denied" response from the server, indicating the firewall was still open.


<img width="1020" height="581" alt="lambda log error" src="https://github.com/user-attachments/assets/b6864a9e-8f8f-416e-bda3-1da053b0c9a2" />

**How did I diagnose the issue?** I opened the CloudWatch Logs for the Lambda function `(/aws/lambda/AutoBlocker)` to read the execution output. I found this specific error message: `Skipped [My IP] (Rule might already exist)`.

<img width="573" height="109" alt="image" src="https://github.com/user-attachments/assets/2d461d12-9944-4db0-b36c-813f4b23962c" />

**What steps were taken to fix it?** I realized that AWS automatically creates a default "Allow All" rule in the Network ACL at Rule #100. Because my Python script was hardcoded to create the DENY rule at `rule_num = 100`, AWS rejected it. I updated the Python script to use `rule_num = 50`. Because Network ACLs evaluate rules in numerical order (lowest to highest), Rule 50 successfully intercepts and blocks the attacker before the default Rule 100 can allow them in. I then updated the Lambda function using `aws lambda update-function-code`.

# Final Validation & Testing

_To prove the automated defense system was fully operational, I conducted an end-to-end test._

1. I rapidly executed 5 failed SSH login attempts from an external machine (my laptop).

2. The CloudWatch Alarm correctly entered the In Alarm state.

3.  The SNS Topic successfully triggered the Lambda function in the background.

4.  When attempting to SSH into the server a final time, the terminal completely froze and returned a Timeout error, proving the network layer had dropped the connection.

<img width="757" height="105" alt="my ip has been blocked" src="https://github.com/user-attachments/assets/091eb733-1f0f-48bd-a1f7-02311ef263cb" />

<img width="992" height="222" alt="ip blocked on the dashboard" src="https://github.com/user-attachments/assets/41dc57c7-3924-4a91-9bed-6bad8825657d" />

<img width="551" height="103" alt="blocked my ip now" src="https://github.com/user-attachments/assets/ff98bf90-c450-44e6-a4f5-44fca7f61a01" />

# Chapter 6: Conclusion & Lessons Learned

This project provided a deep dive into the practical application of the AWS Well-Architected Framework, specifically within the Security and Operational Excellence pillars. 

### Key Takeaways:

1. **The Reality of Cloud Security:** By analyzing the real-world logs in CloudWatch, I saw firsthand that any public-facing server is a constant target for automated brute-force attacks. Within minutes of launching the server, "Invalid user" login attempts from global IP addresses were already being recorded. This highlighted the critical importance of having a security layer active from day one.

2. **The Power of Simple Configurations:** I learned that high-level security doesn't always require complex, expensive software. By simply configuring an OS-level logging agent (CloudWatch Agent), setting up a metric filter, and adjusting VPC-level firewall rules (Network ACLs), I was able to create a professional-grade defense system.

3. **Visibility via Monitoring:** Before this project, I viewed logs as "boring text files." I now understand that logs are the "eyes" of a cloud architect. Without the CloudWatch Alarm, an attacker could attempt thousands of logins without the administrator ever knowing. Adding an alarm transforms passive data into an active defense mechanism.

4. **Automation (DevSecOps):** The transition from an IDS (Detection) to an IPS (Prevention) showed me how powerful automation is. By using a small Lambda function, I removed the "human bottleneck." The system now defends itself 24/7, blocking threats in seconds—far faster than any human administrator could ever react.

5. **Infrastructure as Code (IaC):** Building this project using CLI scripts changed how I view deployment. Being able to destroy my entire lab to save budget and then "resurrect" the entire infrastructure in under a minute proved that IaC is the only way to manage modern, scalable cloud environments.

### Final Thought
Security is not a one-time setup; it is a continuous process of monitoring, alerting, and automated responding. This project has given me the foundational skills to build, defend, and govern secure cloud workloads.


## Sources
- <a href="https://docs.aws.amazon.com/cli/latest/">AWS CLI commands</a>
- <a href="https://docs.aws.amazon.com/">AWS Docs</a>
- <a href="https://awsacademy.instructure.com/courses/149546/modules">AWS Academy Materials</a>
