# AWS-Highly-Available-Web-Infrastructure-with-Custom-VPC-ALB-and-ASG
This project demonstrates a production-ready, fault-tolerant, and highly available web infrastructure deployed entirely on AWS. It showcases core network isolation techniques, automated traffic balancing, and auto-scaling capabilities.

##  Architecture Overview

The infrastructure spans across multiple Availability Zones (Multi-AZ) to eliminate single points of failure.

*   **Network Isolation:** Designed a custom VPC with 2 Public Subnets (to host the Load Balancer) and 2 Private Subnets (to isolate the EC2 application servers from the public internet).
*   **NAT Gateway Integration:** Deployed a NAT Gateway inside the public subnet and configured routing for private subnets, enabling safe internet access for EC2 instances to download software patches without exposing them to direct inbound internet traffic.
*   **Traffic Management:** Configured an Internet-facing Application Load Balancer (ALB) to distribute web traffic evenly across backend servers with automated target group health checks.
*   **Elastic Scaling:** Set up an Auto Scaling Group (ASG) linked with a Launch Template utilizing IMDSv2 metadata services to dynamically maintain server count based on CPU tracking policies.

---

## 🛠️ Step-by-Step Implementation Detail

### 1. Network & Routing Setup
*   **VPC Created:** `10.0.0.0/16`
*   **Subnets:** Created 4 distinct subnets across two AZs (`10.0.1.0/24` & `10.0.2.0/24` as Public; `10.0.3.0/24` & `10.0.4.0/24` as Private).
*   **Internet Access:** Attached an Internet Gateway (IGW) to the VPC. Associated public subnets with a Route Table pointing `0.0.0.0/0` to the IGW.
*   **Private Internet (NAT):** Provisioned a NAT Gateway with an Elastic IP inside a Public Subnet. Bound a secondary Route Table to the Private Subnets routing `0.0.0.0/0` traffic through the NAT Gateway.

### 2. Least-Privilege Security Group Architecture
*   **ALB-SG:** Allowed inbound **HTTP (Port 80)** traffic from anywhere (`0.0.0.0/0`).
*   **EC2-SG:** Bound to backend web servers. Restricted inbound HTTP (Port 80) access **exclusively to the ALB-SG Security Group ID**. Direct internet access to instances is fully blocked.

### 3. Load Balancing & Target Group Configuration
*   Deployed an **Internet-facing Application Load Balancer** mapped across both Public Subnets.
*   Created a Target Group configured for EC2 Instances on Port 80 with standard HTTP health checks directed at the root path (`/`).
### 4. Dynamic Auto Scaling Group Setup
*   **Launch Template Configuration:** Created a template using Amazon Linux 2023 (`t2.micro`). Configured **IMDSv2** with a Metadata Response Hop Limit of `2` to securely fetch EC2 metadata behind the ALB.
*   **User Data Bootstrap Script:**
    ```bash
    #!/bin/bash
    sudo dnf update -y
    sudo dnf install -y httpd
    sudo systemctl start httpd
    sudo systemctl enable httpd

    TOKEN=\$(curl -s -X PUT "http://169.254.169.254" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
    INSTANCE_ID=(curl -s -H "X-aws-ec2-metadata-token: TOKEN" http://169.254.169.254)
    AZ=(curl -s -H "X-aws-ec2-metadata-token: TOKEN" http://169.254.169.254)

    echo "<h1>Welcome to My Scalable Web App</h1>" > /var/www/html/index.html
    echo "<p>Served from Instance ID: <b>\(INSTANCE_ID</b> in Availability Zone: <b>\)AZ</b></p>" >> /var/www/html/index.html
    ```
*   **ASG Metrics:** Set up with Desired: 2, Min: 2, Max: 4 sizing capacity. Implemented a Target Tracking Scaling Policy targeting an average CPU utilization of 60%.

---

## 🧪 Testing and Validation

1.  **Load Balancing Verification:** Hit the ALB DNS endpoint multiple times on a browser. Confirmed that traffic alternated and loaded dynamic content showing varying Instance IDs and AZ parameters from the bootstrap output.
2.  **High Availability / Auto-Healing:** Manually terminated one EC2 instance from the console. Observed the Target Group health tracking mark it unhealthy, while the ASG immediately provisions a new replacement server automatically to restore the desired state.

