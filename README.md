# Lab M3.04 - Deploy 3-Tier Application Architecture

**Repository:** [https://github.com/cloud-engineering-bootcamp/ce-lab-three-tier-application](https://github.com/cloud-engineering-bootcamp/ce-lab-three-tier-application)

**Activity Type:** Individual  
**Estimated Time:** 60-90 minutes

## Learning Objectives

- [ ] Design and implement complete 3-tier architecture
- [ ] Deploy presentation, application, and data tiers
- [ ] Configure security groups for tier isolation
- [ ] Implement proper network segmentation
- [ ] Document architecture decisions and trade-offs
- [ ] Understand traffic flow between tiers

## Your Task

Build a complete 3-tier application architecture on AWS:
1. **Tier 1 (Presentation):** Load Balancer in public subnets
2. **Tier 2 (Application):** Web/App servers in private subnets
3. **Tier 3 (Data):** Database placeholder in private subnets
4. Configure security groups for proper isolation
5. Document architecture with diagrams

**Success Criteria:** Each tier is isolated, traffic flows correctly through all tiers.

## 📤 What to Submit

**Submission Type:** GitHub Repository

Create a **public** GitHub repository named `ce-lab-three-tier-architecture` containing:

### Required Files

**1. README.md**
- Architecture overview and design decisions
- Why 3-tier architecture?
- Security considerations
- Traffic flow explanation
- Future improvements

**2. Architecture Documentation** (`architecture/` folder)
- `ARCHITECTURE.md` - Detailed architecture description
- `SECURITY.md` - Security group design and rationale
- `NETWORK-DESIGN.md` - Subnet and routing design
- `architecture-diagram.png` - Visual architecture diagram

**3. Configuration Files** (`config/` folder)
- `vpc-subnets.txt` - VPC and subnet configuration
- `security-groups.txt` - All security group rules
- `instances.txt` - EC2 instances in each tier
- `load-balancer.txt` - ALB configuration

**4. Application Code** (`app/` folder)
- Web server code (Tier 2)
- Database simulation script (Tier 3)
- Health check endpoints

**5. Testing Documentation** (`tests/` folder)
- `traffic-flow-test.md` - Tier-to-tier connectivity tests
- `security-test.md` - Security isolation verification
- `end-to-end-test.md` - Full application flow test

## Grading: 100 points

- Complete 3-tier architecture: 30pts
- Proper security group isolation: 25pts
- Network design and routing: 20pts
- Documentation and diagrams: 15pts
- Testing and validation: 10pts

## Architecture Design

### Target Architecture

```
┌─────────────────────────────────────────────────────────┐
│ INTERNET                                                 │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│ TIER 1: PRESENTATION (Public Subnets)                   │
│                                                          │
│  AZ-A (10.0.1.0/24)         AZ-B (10.0.2.0/24)         │
│     └── ALB Node                └── ALB Node            │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│ TIER 2: APPLICATION (Private Subnets - App)             │
│                                                          │
│  AZ-A (10.0.11.0/24)        AZ-B (10.0.12.0/24)        │
│     └── App Servers (2)         └── App Servers (2)     │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│ TIER 3: DATA (Private Subnets - Database)               │
│                                                          │
│  AZ-A (10.0.21.0/24)        AZ-B (10.0.22.0/24)        │
│     └── DB Instance             └── DB Standby          │
└─────────────────────────────────────────────────────────┘
```

## Detailed Instructions

### Part 1: Plan Architecture (15 min)

**Subnet Planning:**
```
VPC: 10.0.0.0/16

Tier 1 - Presentation (Public):
- public-subnet-1a:  10.0.1.0/24
- public-subnet-1b:  10.0.2.0/24

Tier 2 - Application (Private):
- app-subnet-1a:     10.0.11.0/24
- app-subnet-1b:     10.0.12.0/24

Tier 3 - Data (Private):
- data-subnet-1a:    10.0.21.0/24
- data-subnet-1b:    10.0.22.0/24
```

**Security Groups:**
```
1. alb-sg         → Allow 80,443 from 0.0.0.0/0
2. app-tier-sg    → Allow 80,443 from alb-sg
3. data-tier-sg   → Allow 3306/5432 from app-tier-sg only
```

### Part 2: Create Additional Subnets (10 min)

**Data Tier Subnets:**
```bash
# Data Subnet AZ-A
DATA_SUBNET_1=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.21.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=data-subnet-1a},{Key=Tier,Value=data}]' \
  --query 'Subnet.SubnetId' --output text)

# Data Subnet AZ-B
DATA_SUBNET_2=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.22.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=data-subnet-1b},{Key=Tier,Value=data}]' \
  --query 'Subnet.SubnetId' --output text)

# Associate with private route table
aws ec2 associate-route-table --route-table-id $PRIVATE_RT --subnet-id $DATA_SUBNET_1
aws ec2 associate-route-table --route-table-id $PRIVATE_RT --subnet-id $DATA_SUBNET_2
```

### Part 3: Create Security Groups (15 min)

**Already have from previous labs:**
- alb-sg
- app-tier-sg (web-servers-sg)

**Create Data Tier Security Group:**
```bash
DATA_SG=$(aws ec2 create-security-group \
  --group-name data-tier-sg \
  --description "Security group for database tier" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

# Allow MySQL from App Tier ONLY
aws ec2 authorize-security-group-ingress \
  --group-id $DATA_SG \
  --protocol tcp --port 3306 --source-group $APP_TIER_SG

# Allow PostgreSQL from App Tier ONLY
aws ec2 authorize-security-group-ingress \
  --group-id $DATA_SG \
  --protocol tcp --port 5432 --source-group $APP_TIER_SG

echo "Data Tier SG: $DATA_SG"
```

**Update App Tier SG for Database Access:**
```bash
# Allow outbound to Data Tier
aws ec2 authorize-security-group-egress \
  --group-id $APP_TIER_SG \
  --protocol tcp --port 3306 --destination-group $DATA_SG

aws ec2 authorize-security-group-egress \
  --group-id $APP_TIER_SG \
  --protocol tcp --port 5432 --destination-group $DATA_SG
```

### Part 4: Deploy Tier 2 (Application) (20 min)

**Application Server Code (Node.js):**
```javascript
// app/server.js
const http = require('http');
const { Client } = require('pg'); // PostgreSQL client

const INSTANCE_ID = process.env.INSTANCE_ID || 'unknown';
const DB_HOST = process.env.DB_HOST || '10.0.21.10';

// Simulate database connection
function queryDatabase() {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve({
        users: 42,
        posts: 156,
        lastUpdate: new Date().toISOString()
      });
    }, 100);
  });
}

const server = http.createServer(async (req, res) => {
  if (req.url === '/health') {
    res.writeHead(200, {'Content-Type': 'application/json'});
    res.end(JSON.stringify({ status: 'healthy', instance: INSTANCE_ID }));
    return;
  }

  if (req.url === '/api/stats') {
    const dbData = await queryDatabase();
    res.writeHead(200, {'Content-Type': 'application/json'});
    res.end(JSON.stringify({
      instance: INSTANCE_ID,
      database: DB_HOST,
      data: dbData
    }));
    return;
  }

  // Main page
  const dbData = await queryDatabase();
  res.writeHead(200, {'Content-Type': 'text/html'});
  res.end(`
    <!DOCTYPE html>
    <html>
    <head>
      <title>3-Tier Application</title>
      <style>
        body { font-family: Arial; padding: 50px; background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); color: white; }
        .container { background: rgba(255,255,255,0.1); padding: 30px; border-radius: 10px; backdrop-filter: blur(10px); }
        .tier { margin: 20px 0; padding: 15px; background: rgba(255,255,255,0.2); border-radius: 5px; }
        .data { color: #ffd700; font-weight: bold; }
      </style>
    </head>
    <body>
      <div class="container">
        <h1>🏗️ 3-Tier Architecture Demo</h1>
        
        <div class="tier">
          <h2>📱 Tier 1: Presentation Layer</h2>
          <p>You're seeing this through the Application Load Balancer</p>
          <p class="data">ALB DNS: [Load Balancer DNS]</p>
        </div>
        
        <div class="tier">
          <h2>⚙️ Tier 2: Application Layer</h2>
          <p>Served by Application Server</p>
          <p class="data">Instance: ${INSTANCE_ID}</p>
        </div>
        
        <div class="tier">
          <h2>💾 Tier 3: Data Layer</h2>
          <p>Database Information:</p>
          <p class="data">Database Host: ${DB_HOST}</p>
          <p class="data">Total Users: ${dbData.users}</p>
          <p class="data">Total Posts: ${dbData.posts}</p>
          <p class="data">Last Update: ${dbData.lastUpdate}</p>
        </div>
        
        <p style="margin-top: 30px; text-align: center; opacity: 0.8;">
          🎓 Cloud Engineering Bootcamp - Week 3 Lab
        </p>
      </div>
    </body>
    </html>
  `);
});

server.listen(80, () => {
  console.log(`App server running (Instance: ${INSTANCE_ID})`);
});
```

**User Data for App Servers:**
```bash
#!/bin/bash
yum update -y
yum install -y nodejs

INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
DB_HOST="10.0.21.10"  # Tier 3 database IP

cat > /home/ec2-user/server.js <<'APPCODE'
[Insert application code above]
APPCODE

export INSTANCE_ID=$INSTANCE_ID
export DB_HOST=$DB_HOST
cd /home/ec2-user
nohup node server.js > app.log 2>&1 &
```

**Launch App Servers:**
```bash
# Launch 2 app servers in AZ-A
for i in 1 2; do
  aws ec2 run-instances \
    --image-id ami-0c55b159cbfafe1f0 \
    --instance-type t2.micro \
    --key-name your-key-pair \
    --security-group-ids $APP_TIER_SG \
    --subnet-id $APP_SUBNET_1 \
    --user-data file://app-userdata.sh \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=app-server-1a-$i},{Key=Tier,Value=app}]"
done

# Launch 2 app servers in AZ-B
for i in 1 2; do
  aws ec2 run-instances \
    --image-id ami-0c55b159cbfafe1f0 \
    --instance-type t2.micro \
    --key-name your-key-pair \
    --security-group-ids $APP_TIER_SG \
    --subnet-id $APP_SUBNET_2 \
    --user-data file://app-userdata.sh \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=app-server-1b-$i},{Key=Tier,Value=app}]"
done
```

### Part 5: Deploy Tier 3 (Data) - Placeholder (15 min)

**Database Placeholder Script:**
```bash
#!/bin/bash
# This simulates a database server
# In production, you'd use RDS, not EC2

yum update -y
yum install -y nc netcat

# Create simple TCP listener on port 3306 (MySQL)
while true; do
  echo -e "HTTP/1.1 200 OK\n\n{\"status\":\"db_healthy\",\"connections\":5}" | nc -l -p 3306
done &

# Create simple TCP listener on port 5432 (PostgreSQL)
while true; do
  echo -e "HTTP/1.1 200 OK\n\n{\"status\":\"db_healthy\",\"connections\":5}" | nc -l -p 5432
done &

echo "Database placeholder running on ports 3306 and 5432"
```

**Launch Database Instance:**
```bash
DB_INSTANCE=$(aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t2.micro \
  --key-name your-key-pair \
  --security-group-ids $DATA_SG \
  --subnet-id $DATA_SUBNET_1 \
  --private-ip-address 10.0.21.10 \
  --user-data file://db-userdata.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=database-1a},{Key=Tier,Value=data}]' \
  --query 'Instances[0].InstanceId' --output text)

echo "Database Instance: $DB_INSTANCE"
```

### Part 6: Integrate with ALB (10 min)

**Update Target Group (if needed):**
```bash
# Deregister old instances
aws elbv2 deregister-targets --target-group-arn $TG_ARN --targets Id=i-old1 Id=i-old2

# Register new app tier instances
APP_INSTANCES=$(aws ec2 describe-instances \
  --filters "Name=tag:Tier,Values=app" "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].InstanceId' --output text)

for instance in $APP_INSTANCES; do
  aws elbv2 register-targets --target-group-arn $TG_ARN --targets Id=$instance
done

# Wait for health checks
sleep 30
aws elbv2 describe-target-health --target-group-arn $TG_ARN
```

### Part 7: Test 3-Tier Architecture (15 min)

**Test 1: End-to-End Flow**
```bash
# Access through ALB
curl http://$ALB_DNS

# Should see:
# - Tier 1: ALB DNS
# - Tier 2: App server instance ID
# - Tier 3: Database data
```

**Test 2: API Endpoint**
```bash
curl http://$ALB_DNS/api/stats

# Expected JSON response with database data
```

**Test 3: Tier Isolation**
```bash
# Try to access database directly from internet (should FAIL)
nc -zv 10.0.21.10 3306
# Expected: Connection timeout or refused

# Try to access app server directly from internet (should FAIL)
curl http://10.0.11.10
# Expected: Timeout (no route from internet)
```

**Test 4: Verify Security Groups**
```bash
# From app server, test database connection
ssh app-server-ip
nc -zv 10.0.21.10 3306
# Expected: Success (app can reach database)

# From app server, test internet access (via NAT)
curl http://checkip.amazonaws.com
# Expected: Success (NAT Gateway public IP)
```

## Architecture Diagram

Create a visual diagram showing:
- All 3 tiers with subnets
- Security group boundaries
- Traffic flow arrows
- IP address ranges
- Components in each tier

**Tools:**
- draw.io
- Lucidchart
- CloudCraft
- AWS Architecture Icons

## Reflection Questions

Answer in `ARCHITECTURE.md`:

1. **Why separate application and database tiers?**

2. **What are the security benefits of this architecture?**

3. **How does traffic flow from internet to database?**

4. **What happens if one availability zone fails?**

5. **How would you scale each tier independently?**

6. **What are the trade-offs of 3-tier vs monolith?**

## Bonus Challenges

**+5 points each:**
- [ ] Add internal load balancer between Tier 2 and Tier 3
- [ ] Implement Auto Scaling for Tier 2
- [ ] Deploy actual RDS database instead of placeholder
- [ ] Add CloudWatch dashboard for all 3 tiers
- [ ] Implement bastion host for SSH access

## Resources

- [3-Tier Architecture Guide](https://docs.aws.amazon.com/whitepapers/latest/serverless-multi-tier-architectures-api-gateway-lambda/three-tier-architecture-overview.html)
- [AWS Reference Architectures](https://aws.amazon.com/architecture/)
- [Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)

---

**Excellent work on building a production-ready 3-tier architecture!** 🏗️
