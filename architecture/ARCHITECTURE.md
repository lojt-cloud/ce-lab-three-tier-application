===================================================
3-TIER ARCHITECTURE DEPLOYMENT SUMMARY
===================================================
VPC ID: $VPC_ID

TIER 1 (Presentation):
- ALB Name: app-alb
- ALB DNS: $ALB_DNS
- Target Group: app-target-group
- Security Group: alb-sg (Inbound 80 from 0.0.0.0/0)

TIER 2 (Application):
- Security Group: app-tier-sg (Inbound 80 from alb-sg)
- App Servers: app-server-1a, app-server-1b
- Private Subnets: 10.0.11.0/24 (AZ-a), 10.0.12.0/24 (AZ-b)

TIER 3 (Data):
- Security Group: db-tier-sg (Inbound 3306 & ICMP from app-tier-sg)
- Database Server: db-server-1a (10.0.21.10)
- Isolated Subnet: 10.0.21.0/24 (AZ-a)
===================================================

# Reflection Questions

# Why separate application and database tiers?
Scalability, compliance, least privilege principle, in case of a security breach the blas radius is smaller. 

### 1. What are the security benefits of this architecture?
This architecture enforces defence-in-depth and reduces the attack surface by ensuring external users and potential attackers can only reach the surface-level Application Load Balancer in the public subnet, completely isolating the application servers and database tier from direct public internet access.

### 2. How does traffic flow from internet to database?
Inbound user traffic flows from the internet through the Internet Gateway (IGW) to the Application Load Balancer in the public subnets, which forwards HTTP requests to the Node.js application servers in the private app subnets, which then securely query the database in the isolated data subnets over TCP port 3306; outbound internet requests (such as software updates) route separately from the app tier through the NAT Gateway in the public subnet.

### 3. What happens if one availability zone fails?
If an entire Availability Zone experiences an outage, the multi-AZ design ensures seamless high availability because the Application Load Balancer automatically detects unhealthy targets via health checks and routes 100% of incoming user traffic to the remaining healthy servers in the active Availability Zone until the failed zone recovers.

### 4. How would you scale each tier independently?
Each tier can be scaled independently according to its specific resource demands: Tier 1 (ALB) scales automatically through managed AWS load balancing, Tier 2 (Application) utilizes Auto Scaling Groups (ASGs) to dynamically add or remove EC2 instances across Availability Zones based on CPU or memory metrics, and Tier 3 (Database) scales vertically by adjusting instance sizes or horizontally by deploying Read Replicas for read-heavy workloads.

### 5. What are the trade-offs of 3-tier vs monolith?
While a monolithic architecture is simpler and cheaper to deploy initially, a single bug or resource crash can bring down the entire application; in contrast, a 3-tier architecture minimises blast radius and allows individual tiers to be recovered or updated independently, though it introduces greater operational complexity, additional networking components, and slight latency between network hops.
