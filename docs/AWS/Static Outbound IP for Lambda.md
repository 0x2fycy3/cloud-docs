# Configuration: Lambda Inside a VPC with Static Outbound IP (Elastic IP)

**Technical document — step-by-step first, followed by observations and troubleshooting notes**

> Objective: Ensure that an AWS Lambda function, when executed inside a VPC, has **Internet access** and that **all outbound traffic** uses a **fixed public IP address (Elastic IP)** — useful for whitelisting external services (e.g., a database that only accepts specific IP addresses).

> Source: https://repost.aws/knowledge-center/lambda-static-ip

---

# STEP-BY-STEP

> Initial Notes: In this example, we use a VPC CIDR block `10.0.0.0/24`. You may use another CIDR block if needed, but adjust subnet ranges accordingly. All steps below use the AWS Console (Infrastructure as Code can also be used if preferred).

---

## 1 — Create the VPC

1. AWS Console → **VPC** → **Your VPCs** → **Create VPC**
2. Name: `bubble-db-migration-vpc` (example)
3. IPv4 CIDR block: `10.0.0.0/24`
4. Create.

---

## 2 — Create Subnets (Minimum: 1 Public + 2 Private for Lambda High Availability)

**Recommended subnet split for `10.0.0.0/24`:** use /26 blocks (64 IPs each).

* **Public Subnet (AZ A)**

  * Name: `public-subnet-1`
  * IPv4 CIDR: `10.0.0.0/26`

* **Private Subnet A (AZ A)**

  * Name: `private-subnet-1`
  * IPv4 CIDR: `10.0.0.64/26`

* **Private Subnet B (AZ B)**

  * Name: `private-subnet-2`
  * IPv4 CIDR: `10.0.0.128/26`

Create each subnet in **different Availability Zones** (when possible) for redundancy.

---

## 3 — Create and Attach an Internet Gateway (IGW)

1. VPC → **Internet Gateways** → **Create Internet Gateway**
2. Name: `bubble-db-migration-igw`
3. Select the IGW → **Actions → Attach to VPC** → choose your VPC.

---

## 4 — Configure the Public Route Table

1. VPC → **Route Tables** → **Create Route Table**

   * Name: `public-route-table`
   * VPC: your VPC

2. Select the route table → **Edit Routes** → **Add Route**

   * Destination: `0.0.0.0/0`
   * Target: your Internet Gateway (`igw-xxxx`)

3. Go to **Subnet Associations** → Associate → select `public-subnet-1`.

Now `public-subnet-1` is truly public.

---

## 5 — Allocate an Elastic IP

1. VPC (or EC2) → **Elastic IPs** → **Allocate Elastic IP Address**
2. Save the allocated EIP (e.g., `34.x.x.x`) — this will be your static outbound IP.

---

## 6 — Create NAT Gateway in Public Subnet and Associate Elastic IP

1. VPC → **NAT Gateways** → **Create NAT Gateway**
2. Subnet: `public-subnet-1`
3. Elastic IP allocation ID: select the allocated EIP
4. Create.

> Result: NAT Gateway `nat-xxxxxxxx` with associated Elastic IP.

**(Production recommendation):** Create one NAT Gateway per AZ and associate separate EIPs for high availability.

---

## 7 — Configure Private Route Table (NAT)

1. VPC → **Route Tables** → **Create Route Table**

   * Name: `private-route-table`
   * VPC: your VPC

2. Edit Routes → **Add Route**

   * Destination: `0.0.0.0/0`
   * Target: your NAT Gateway (`nat-xxxxx`)

3. Go to **Subnet Associations**

   * Associate `private-subnet-1`
   * Associate `private-subnet-2`

Now all traffic from private subnets exits through the NAT Gateway and uses the Elastic IP.

---

## 8 — Create Security Group for Lambda

1. VPC → **Security Groups** → **Create Security Group**

   * Name: `lambda-sg`
   * Description: `Lambda VPC SG`
   * VPC: your VPC

**Recommended Rules:**

* **Outbound (Egress):** Allow All Traffic → `0.0.0.0/0`
* **Inbound (Ingress):** None required (unless Lambda must receive traffic)

---

## 9 — Prepare IAM Role for Lambda (Execution Role)

Lambda must have permission to create ENIs inside the VPC.

### Option A — Recommended (Simplest)

* IAM → **Roles**
* Select the execution role used by your Lambda (or create a new one)
* Attach managed policy:
  **AWSLambdaVPCAccessExecutionRole**

---

### Option B — Custom Policy (Minimum Required)

Attach the following inline policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect":"Allow",
      "Action":[
        "ec2:CreateNetworkInterface",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DeleteNetworkInterface",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeVpcs",
        "ec2:AssignPrivateIpAddresses",
        "ec2:UnassignPrivateIpAddresses"
      ],
      "Resource":"*"
    }
  ]
}
```

After attaching the policy, wait a few seconds and retry saving the VPC configuration in Lambda.

---

## 10 — Configure Lambda to Use the VPC

1. Lambda Console → Your function → **Configuration → VPC (or Network) → Edit**
2. Select your VPC (`10.0.0.0/24`)
3. Select both private subnets (`private-subnet-1` and `private-subnet-2`)
4. Select Security Group `lambda-sg`
5. Save.

AWS will create ENIs in the private subnets.

---

## 11 — Test From Lambda (Python + urllib3 Example)

```python
import json
import urllib3

def lambda_handler(event, context):
    http = urllib3.PoolManager()
    resp = http.request('GET', 'https://ifconfig.me')
    ip = resp.data.decode('utf-8') if resp.status == 200 else None

    return {
        "statusCode": 200,
        "body": json.dumps({
            "public_ip": ip,
            "status": resp.status
        })
    }
```

Expected result:
`public_ip` must match the Elastic IP assigned to the NAT Gateway.

---

# Quick Summary

* VPC creation
* 1 Public subnet + ≥2 Private subnets (different AZs)
* Internet Gateway created and attachment
* Public Route Table: `0.0.0.0/0 → IGW`
* Elastic IP allocated
* NAT Gateway created in public subnet with EIP
* Private Route Table: `0.0.0.0/0 → NAT Gateway`
* Security Group created for Lambda
* IAM role includes `AWSLambdaVPCAccessExecutionRole`
* Lambda configured with private subnets and SG
* Test confirms outbound IP equals Elastic IP

---

# TROUBLESHOOTING & LESSONS LEARNED (Based on Real Issues I Encountered)

### 1. Subnet CIDR Overlap

**Error:**

```
CIDR Address overlaps with existing Subnet CIDR: 10.0.0.0/24
```

**Cause:**
A subnet was created using the full VPC CIDR (`/24`), leaving no room for additional subnets.

**Fix:**
Delete the large subnet and recreate smaller, non-overlapping subnets (e.g., `/26`).

---

### 2. Public Route Table Missing Internet Route

**Symptom:**
Route table only had:

```
10.0.0.0/24 → local
```

**Fix:**
Add:

```
0.0.0.0/0 → Internet Gateway
```

---

### 3. Private Route Table Missing NAT Route

**Symptom:**
Lambda could not access internet.

**Fix:**
Add:

```
0.0.0.0/0 → NAT Gateway
```

---

### 4. IAM Role Missing ENI Permissions

**Error:**

```
The provided execution role does not have permissions to call CreateNetworkInterface on EC2
```

**Fix:**
Attach `AWSLambdaVPCAccessExecutionRole`.

---

### 5. Incorrect Security Group

**Fix:**
Create dedicated SG with outbound `0.0.0.0/0`.

---

### 6. Cost Awareness

NAT Gateway has hourly + data processing charges.
For low-cost environments, a NAT Instance (EC2) is an alternative, but requires management.

---

### 7. High Availability

For production:

* Use multiple private subnets in different AZs
* Use one NAT Gateway per AZ

---

### 8. Lambda Scaling & ENIs

When Lambda scales, it creates ENIs in selected subnets.
Ensure enough available IP addresses per subnet.
