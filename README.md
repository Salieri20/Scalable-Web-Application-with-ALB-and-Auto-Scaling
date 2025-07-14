# Scalable 2-Tier Web Application on AWS

This project demonstrates how to deploy a highly available and scalable 2-tier web application on AWS using core services like EC2, ALB, ASG, and RDS.

## Architecture Overview

![Architecture Diagram](aws-architecture.png)

## Key AWS Services Used

This architecture represents a 2-tier scalable web application deployed on AWS using the following services:

- **EC2**: Hosts the web application across multiple instances.
- **Application Load Balancer (ALB)**: Distributes incoming traffic to healthy EC2 instances.
- **Auto Scaling Group (ASG)**: Automatically adds or removes EC2 instances based on load.
- **Amazon RDS (MySQL)**: Managed relational database with Multi-AZ for high availability.
- **IAM**: Used for assigning least-privilege access to services.
- **CloudWatch + SNS**: Monitors metrics and sends notifications on specific events.
- **NAT Gateway**: Allows instances in private subnets to access the internet securely in case the EC2 instances need to be updated or batched.
- **Bastion Host**: Deployed in a public subnet to access private EC2 instances via SSH.



## IAM Roles and Policies

To ensure secure and fine-grained access control across AWS services in this project, several IAM roles with appropriate managed policies were created and assigned to the respective services:

### EC2 Instance Role (Web Tier)
**Role Name:** `WebTierEC2Role`  
**Attached Policy:**  
- `AmazonSSMManagedInstanceCore` – Enables Systems Manager access (e.g., Session Manager).
- `CloudWatchAgentServerPolicy` – Allows publishing custom metrics and logs to CloudWatch.

**Used By:** EC2 instances in the Web Tier (behind ALB).  
**Purpose:**  
- Allow EC2 to be managed securely without SSH.
- Push logs and monitoring metrics to CloudWatch.

---

### ALB Role (Application Load Balancer)
**Role Name:** `ALBAccessRole`  
**Attached Policy:**  
- `AWSElasticLoadBalancingServiceRolePolicy` – Grants ALB permission to manage health checks and register instances.

**Used By:** ALB (public-facing).  
**Purpose:**  
- Allow ALB to interact with EC2 instances for health checking and traffic routing.

---

### Auto Scaling Group Role
**Role Name:** `ASGExecutionRole`  
**Attached Policy:**  
- `AWSServiceRoleForAutoScaling` – Allows Auto Scaling to launch, terminate, and manage EC2 instances.

**Used By:** Auto Scaling Groups for the Web Tier.  
**Purpose:**  
- Enable Auto Scaling lifecycle operations.

---

### CloudWatch Monitoring Role
**Role Name:** `CloudWatchMonitoringRole`  
**Attached Policy:**  
- `CloudWatchReadOnlyAccess`  
- `CloudWatchEventsFullAccess` (optional for custom alarms/events)

**Used By:** CloudWatch alarms and metrics.  
**Purpose:**  
- Monitor EC2 performance and ASG activities.
- Trigger notifications via SNS.

---

### SNS Notifications Role
**Role Name:** `SNSNotifierRole`  
**Attached Policy:**  
- `AmazonSNSFullAccess`

**Used By:** CloudWatch to publish alerts.  
**Purpose:**  
- Send notification emails to the administrator when critical events occur (e.g., instance failure, scaling activity, RDS failover).

---

### Bastion Host Role (Optional)
**Role Name:** `BastionAccessRole`  
**Attached Policy:**  
- `AmazonSSMManagedInstanceCore` – For Systems Manager access.
- `EC2InstanceConnect` (if using EC2 Connect for SSH)

**Used By:** Bastion host instance.  
**Purpose:**  
- Secure access point to private subnet resources.

---

> *All roles are granted least-privilege permissions aligned with the principle of least privilege (PoLP). Custom policies can be defined for finer control if needed.*

---

## Security Groups Configuration

To control traffic flow between different components of the architecture, the following Security Groups were created:

---

### ALB Security Group (`ALB-SG`)
**Inbound Rules:**
- HTTP (80) from `0.0.0.0/0`
- HTTPS (443) from `0.0.0.0/0`

**Outbound Rules:**
- HTTP/HTTPS to Web Tier EC2 Subnet CIDRs

**Purpose:**  
Allow public internet traffic to reach the Web Tier via the Application Load Balancer.

---

### Web Tier EC2 Security Group (`WebTierEC2-SG`)
**Inbound Rules:**
- HTTP (80) and HTTPS (443) from ALB Security Group
- SSH (22) from Bastion Host Security Group (for admin access)

**Outbound Rules:**
- MySQL (3306) to RDS Security Group

**Purpose:**  
Allow traffic from the ALB and provide controlled SSH access for administrative operations. Also allows DB queries to the RDS.

---

### RDS Security Group (`RDS-SG`)
**Inbound Rules:**
- MySQL (3306) from Web Tier EC2 Security Group

**Outbound Rules:**
- (Default — no outbound rules required)

**Purpose:**  
Restrict access to the database so that only EC2 instances in the Web Tier can connect.

---

### Bastion Host Security Group (`BastionHost-SG`)
**Inbound Rules:**
- SSH (22) from your own IP (`xx.xx.xx.xx/32`)

**Outbound Rules:**
- SSH (22) to Web Tier EC2 Security Group

**Purpose:**  
Used as a secure jump host to access EC2 instances in private subnets for maintenance or troubleshooting.

---

> All security groups follow the **least-privilege** model. No direct access to the database or EC2 instances is allowed from the public internet.


## 📈 Project Workflow

1. **User Access via Route 53 & ALB**  
   A user sends an HTTP/HTTPS request to the application's domain name, managed by **Route 53**.

2. **Routing Through Internet Gateway to ALB**  
   Route 53 resolves the domain name and forwards the request through the **Internet Gateway** to the **Application Load Balancer (ALB)**, which is deployed in **public subnets** across multiple AZs.

3. **Load Balancer to Web Tier EC2 Instances**  
   The ALB distributes traffic to the **EC2 instances** that are part of an **Auto Scaling Group (ASG)** in the web tier (also within public subnets). These instances run the actual web application.

4. **Auto Scaling Group Behavior**  
   The ASG automatically adjusts the number of EC2 instances:
   - **Scales out** when CPU utilization or network traffic increases.
   - **Scales in** when demand drops to optimize cost.

5. **Web Tier to Database Access**  
   The EC2 instances interact with the **Amazon RDS database** (MySQL/PostgreSQL), which is deployed in **private subnets** across multiple Availability Zones with **Multi-AZ enabled**.

6. **Admin Access via Bastion Host**  
   A **Bastion Host** is deployed in a **public subnet**, allowing secure **SSH access** to private resources (like RDS or internal EC2s) using **key-based authentication**. This host is only accessible from trusted IP addresses.

7. **Software Updates via NAT Gateway**  
   The EC2 instances are allowed to connect to the internet **outbound** for software updates via a **NAT Gateway**, without being exposed to inbound internet traffic.

8. **CloudWatch Monitoring**  
   **Amazon CloudWatch** tracks key metrics and events:
   - EC2 instance health/status
   - Auto Scaling actions (e.g., launch/terminate)
   - RDS failover events

9. **Alerting via SNS**  
   Critical events are pushed to an **Amazon SNS Topic**.
   - The **Administrator** is subscribed to the topic.
   - They receive real-time **email notifications** on important system changes, such as ASG activities or RDS failover.

10. **High Availability & Fault Tolerance**  
    - The architecture spans **multiple AZs** for redundancy.
    - **Health checks** on the ALB ensure traffic only reaches healthy instances.
    - **Multi-AZ RDS** provides automatic failover to a standby DB in case of failure.

> This architecture ensures scalability, high availability, cost optimization, centralized monitoring, and secure administrative access — all built using AWS best practices.


## Monitoring & Alerts

This project uses Amazon CloudWatch and Amazon SNS to monitor the health and performance of the deployed 2-tier web application on AWS.

---

### EC2 Monitoring

- **Web Tier EC2 Instances**:
  - Monitored for **CPU utilization** to detect load spikes.
  - A CloudWatch alarm (`WebHighCPUAlarm`) is configured to trigger if CPU usage exceeds **75%** for more than **5 minutes**.
  - This alarm is tied to an Auto Scaling policy to automatically increase the number of instances when needed.

---

### Auto Scaling Group (ASG)

- CloudWatch alarms are configured to track:
  - **Instance Launch/Termination Failures**
  - **Scaling Events** (e.g., scale-out or scale-in operations)
- Scaling is driven by average CPU utilization across the EC2 instances in the Web Tier.

---

### Amazon RDS (MySQL)

- Configured in **Multi-AZ** mode for high availability.
- Monitored for:
  - **CPU Utilization** — `RDSHighCPUAlarm` triggers at **80%** usage.
  - **Free Storage Space** — `RDSSpaceLowAlarm` triggers if available space drops below **1 GB**.
  - **Failover Events** — Detected by CloudWatch when the DB automatically switches to standby in case of failure.

---

### SNS Notifications

- All alarms are connected to a dedicated **SNS Topic**:  
  `arn:aws:sns:REGION:ACCOUNT_ID:WebAppOpsAlerts`

- The **admin email** is subscribed to this topic to receive immediate alerts.

#### Example Alerts:
- “Web EC2 CPU > 75%”
- “RDS failover detected”
- “ASG: Failed to launch instance”

---

### Alert Flow Diagram

```mermaid
graph TD
    A[Web EC2 CPU > 75%] --> B(CloudWatch Alarm)
    B --> C{SNS Topic: WebAppOpsAlerts}
    C --> D[Email Notification]

> All monitoring and alerting components are configured with cost-efficiency in mind — only critical metrics are tracked to avoid unnecessary CloudWatch charges.



## Credits

This project was designed, implemented, and documented by **Mahmoud Nasser** as part of the graduation project of the Manara becoming an AWS Solutions Architect program.

- 👨‍💻 **Role**: DevOps Engineer 
- 🌍 **Focus**: Scalable infrastructure on AWS, automation, and cloud-native solutions. 

