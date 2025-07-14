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
- **IAM**: Used for assigning least-privilege access to services (e.g., EC2, ALB, CloudWatch).
- **CloudWatch + SNS**: Monitors metrics and sends notifications on specific events.
- **NAT Gateway**: Allows instances in private subnets to access the internet securely.
- **Bastion Host**: Deployed in a public subnet to access private EC2 instances via SSH.


## IAM Roles and Security

### IAM Roles

- **ALB**: `AWSElasticLoadBalancingServiceRolePolicy`
- **ASG**: `AWSServiceRoleForAutoScaling`
- **EC2 Instances**: Custom role with minimal permissions (e.g., CloudWatchAgentServerPolicy if using monitoring)
- **CloudWatch**: `CloudWatchReadOnlyAccess` (or full access for logs/metrics if needed)

---

### Security Groups

| Resource        | Inbound Rules                            | Outbound Rules                           |
|----------------|-------------------------------------------|-------------------------------------------|
| **ALB**         | `0.0.0.0/0` on ports `80`, `443`          | Web Tier EC2 subnet range on ports `80`, `443` |
| **Web EC2**     | From ALB on ports `80`, `443`             | To RDS on port `3306`                     |
| **RDS (MySQL)** | From Web EC2 subnet range on port `3306`  | —                                         |
| **Bastion Host**| From your IP (SSH) on port `22`           | To private subnet (for SSH access)        |

> *Each tier is protected using subnet segmentation and strict security group rules to minimize attack surface.*


## Project Workflow
1. **A user opens the website** using a custom domain name (e.g., `www.example.com`), which is managed through **Route 53**.

2. **Route 53** resolves the domain name to the **Application Load Balancer (ALB)**'s public DNS name.

3. The request flows through the **Internet Gateway** and hits the **public ALB** in the **web tier**.

4. The **ALB** distributes the request across multiple EC2 instances running in an **Auto Scaling Group (ASG)**. These EC2 instances are deployed in **private subnets** across multiple Availability Zones to ensure high availability.

5. Based on traffic demand:
   - If load increases, the **ASG scales out** by launching more instances.
   - If load decreases, the **ASG scales in** by terminating instances.

6. The **web tier EC2 instance** handles the application logic and connects directly to the **Amazon RDS** database (MySQL/PostgreSQL), which is hosted in **private subnets**.

7. The **RDS instance** is configured in **Multi-AZ deployment mode**, providing high availability and automatic failover in case the primary instance becomes unavailable.

8. **CloudWatch** monitors the following:
   - Instance-level metrics (CPU, memory, etc.)
   - ASG events (instance launch/termination/failures)
   - RDS failover events

9. Upon detecting specific events or alarms, **CloudWatch** publishes notifications to an **SNS Topic**.

10. The **SNS Topic** has the admin subscribed (e.g., via email), so the admin receives real-time alerts.

11. In the event of maintenance, debugging, or emergencies, the **admin accesses private EC2 instances** through a **bastion host** that resides in a **public subnet**. The bastion host uses SSH key pair (with tight security group rules)

> This architecture ensures scalability, high availability, cost optimization, centralized monitoring, and secure administrative access — all built using AWS best practices.


## Monitoring & Alerts

This project uses Amazon CloudWatch and SNS to monitor the health and performance of the deployed 2-tier web application.

### EC2 Monitoring

- **Web Tier EC2 Instances (behind the ALB):**
  - Monitored for CPU utilization to detect unusual load.
  - A CloudWatch alarm (`WebHighCPUAlarm`) is set to trigger if CPU utilization exceeds 75% for more than 5 minutes.
  - Connected to an Auto Scaling policy to scale out if needed.

- **Bastion Host (Jumpbox in Public Subnet):**
  - Basic monitoring enabled (CPU, network), but no scaling or alarms since it’s only used for admin SSH access.

### Auto Scaling Group (ASG)

- Alarms are configured to detect the following:
  - **Launch/Termination Failures** (e.g., failed instance creation due to lack of capacity).
  - **Minimum/Maximum Instance Breach** (e.g., if instance count goes beyond limits).
  - Scaling policies are tied to average CPU utilization on the EC2s.

### ALB (Application Load Balancer)

- Tracks:
  - **HTTP 5XX Errors** – triggers an alarm (`ALB5XXAlarm`) if more than 1% of requests fail over a 5-minute window.
  - **Target Unhealthy Count** – detects if EC2 instances in the target group become unhealthy.

### Amazon RDS

- Monitors:
  - **CPU Utilization**, **Free Storage Space**, and **Database Connections**.
  - Alarm (`RDSHighCPUAlarm`) triggers if CPU exceeds 80%.
  - Alarm (`RDSSpaceLowAlarm`) triggers if available storage drops below 1GB.
  - Multi-AZ is enabled for automatic failover — CloudWatch detects this failover event.

### SNS Notifications

- All the alarms mentioned above are wired to a **dedicated SNS Topic (`WebAppOpsAlerts`)**.
- The **admin email** is subscribed to the topic.
- Example alerts received:
  - “Web Tier EC2 CPU utilization > 75%”
  - “ALB: High 5XX error rate detected”
  - “RDS: Failover occurred”
  - “ASG: Instance launch failure”

### Example Alert Flow

1. A sudden spike in user traffic increases CPU on Web EC2s.
2. `WebHighCPUAlarm` is triggered.
3. Alarm publishes a message to `WebAppOpsAlerts` SNS topic.
4. Admin receives an email:  
   `"ALERT: Web EC2 CPU over 75% for 5 minutes. Consider scaling."`

> All monitoring and alerting components are configured with cost-efficiency in mind — only critical metrics are tracked to avoid unnecessary CloudWatch charges.



## Credits

This project was designed, implemented, and documented by **Mahmoud Nasser** as part of the graduation project of the Manara becoming an AWS Solutions Architect program.

- 👨‍💻 **Role**: DevOps Engineer 
- 🌍 **Focus**: Scalable infrastructure on AWS, automation, and cloud-native solutions. 

