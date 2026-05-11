# Manual AWS Console Steps From Scratch

Follow the PDF architecture: public subnets for the load balancer, private subnets for the backend web servers.

Changes for your setup:

- No Terraform.
- You are on a MacBook, so use AWS Console **Connect** where possible.
- No NAT Gateway, because KodeKloud Studio may not allow it.
- Since there is no NAT Gateway, private instances cannot download packages or clone GitHub. We use built-in Python from Amazon Linux user data to serve simple pages.

## 1. Create VPC

Go to **VPC > Your VPCs > Create VPC**.

| Setting | Value |
| --- | --- |
| Name | `path-routing-vpc` |
| IPv4 CIDR | `10.0.0.0/16` |

## 2. Create Public Subnets

Go to **VPC > Subnets > Create subnet**.

Create two public subnets in different Availability Zones.

| Name | Availability Zone | CIDR | Purpose |
| --- | --- | --- | --- |
| `public-subnet-1` | first AZ | `10.0.1.0/24` | ALB |
| `public-subnet-2` | second AZ | `10.0.2.0/24` | ALB |

After creating them, select each public subnet and enable:

**Actions > Edit subnet settings > Enable auto-assign public IPv4 address**

## 3. Create Private Subnets

Create two private subnets in different Availability Zones.

| Name | Availability Zone | CIDR | Purpose |
| --- | --- | --- | --- |
| `private-subnet-1` | first AZ | `10.0.11.0/24` | AWS backend |
| `private-subnet-2` | second AZ | `10.0.12.0/24` | Azure/GCP backend |

Do not enable public IPv4 assignment on private subnets.

## 4. Create and Attach Internet Gateway

Go to **VPC > Internet gateways > Create internet gateway**.

| Setting | Value |
| --- | --- |
| Name | `path-routing-igw` |

Then select it and choose:

**Actions > Attach to VPC > path-routing-vpc**

## 5. Create Public Route Table

Go to **VPC > Route tables > Create route table**.

| Setting | Value |
| --- | --- |
| Name | `public-rt` |
| VPC | `path-routing-vpc` |

Open `public-rt`, go to **Routes > Edit routes**, and add:

| Destination | Target |
| --- | --- |
| `0.0.0.0/0` | `path-routing-igw` |

Then go to **Subnet associations > Edit subnet associations** and select:

- `public-subnet-1`
- `public-subnet-2`

## 6. Private Route Table

Leave the private subnets associated with the default local route table.

Do not add `0.0.0.0/0` for the private subnets because we are not using NAT Gateway.

## 7. Create Security Groups

Go to **EC2 > Security Groups > Create security group**.

### ALB Security Group

Create:

| Setting | Value |
| --- | --- |
| Name | `path-alb-sg` |
| VPC | `path-routing-vpc` |

Inbound rule:

| Type | Source |
| --- | --- |
| HTTP 80 | `0.0.0.0/0` |

Keep outbound as **Allow all**.

### Private Web Server Security Group

Create:

| Setting | Value |
| --- | --- |
| Name | `private-web-sg` |
| VPC | `path-routing-vpc` |

Inbound rules:

| Type | Source |
| --- | --- |
| HTTP 80 | `path-alb-sg` |
| SSH 22 | EC2 Instance Connect Endpoint security group, if available |

Keep outbound as **Allow all**.

### EC2 Instance Connect Endpoint Security Group

Only create this if KodeKloud allows EC2 Instance Connect Endpoint.

| Setting | Value |
| --- | --- |
| Name | `eic-endpoint-sg` |
| VPC | `path-routing-vpc` |

Outbound rule:

| Type | Destination |
| --- | --- |
| SSH 22 | `private-web-sg` |

## 8. Optional: Create EC2 Instance Connect Endpoint

This is the best replacement for connecting to private instances from your MacBook without a public IP.

Go to **VPC > Endpoints > Create endpoint**.

Use:

| Setting | Value |
| --- | --- |
| Name | `path-routing-eic-endpoint` |
| Service category | EC2 Instance Connect Endpoint |
| VPC | `path-routing-vpc` |
| Subnet | `private-subnet-1` or `private-subnet-2` |
| Security group | `eic-endpoint-sg` |

After it is ready, use:

**EC2 > Instances > Select private instance > Connect > EC2 Instance Connect**

If KodeKloud does not allow this endpoint, you can still finish the lab because user data starts the web servers automatically.

## 9. Launch Private EC2 Instances

Go to **EC2 > Instances > Launch instance**.

Launch three Amazon Linux 2023 instances.

| Instance name | Subnet | Security group | Public IP |
| --- | --- | --- | --- |
| `aws-private-server` | `private-subnet-1` | `private-web-sg` | Disabled |
| `azure-private-server` | `private-subnet-2` | `private-web-sg` | Disabled |
| `gcp-private-server` | `private-subnet-2` | `private-web-sg` | Disabled |

Use:

- AMI: Amazon Linux 2023
- Instance type: `t2.micro` or `t3.micro`
- Auto-assign public IP: Disabled

### User Data for AWS Server

Paste this into **Advanced details > User data** for `aws-private-server`:

```bash
#!/bin/bash
mkdir -p /var/www/html/aws
cat > /var/www/html/aws/index.html <<'HTML'
<h1>Amazon Web Services (AWS)</h1>
<p>This page is served from the private AWS backend instance.</p>
HTML
cd /var/www/html
nohup python3 -m http.server 80 > /tmp/aws-web.log 2>&1 &
```

### User Data for Azure Server

Paste this into **Advanced details > User data** for `azure-private-server`:

```bash
#!/bin/bash
mkdir -p /var/www/html/azure
cat > /var/www/html/azure/index.html <<'HTML'
<h1>Microsoft Azure</h1>
<p>This page is served from the private Azure backend instance.</p>
HTML
cd /var/www/html
nohup python3 -m http.server 80 > /tmp/azure-web.log 2>&1 &
```

### User Data for GCP Server

Paste this into **Advanced details > User data** for `gcp-private-server`:

```bash
#!/bin/bash
mkdir -p /var/www/html/gcp
cat > /var/www/html/gcp/index.html <<'HTML'
<h1>Google Cloud Platform (GCP)</h1>
<p>This page is served from the private GCP backend instance.</p>
HTML
cd /var/www/html
nohup python3 -m http.server 80 > /tmp/gcp-web.log 2>&1 &
```

## 10. Create Target Groups

Go to **EC2 > Target Groups > Create target group**.

Create:

| Target group | Registered instance | Health check path |
| --- | --- | --- |
| `aws-tg` | `aws-private-server` | `/aws/` |
| `azure-tg` | `azure-private-server` | `/azure/` |
| `gcp-tg` | `gcp-private-server` | `/gcp/` |

For each target group, use:

- Target type: Instances
- Protocol: HTTP
- Port: 80
- VPC: `path-routing-vpc`

Wait until the targets show **healthy**.

## 11. Create Application Load Balancer

Go to **EC2 > Load Balancers > Create load balancer > Application Load Balancer**.

| Setting | Value |
| --- | --- |
| Name | `path-routing-alb` |
| Scheme | Internet-facing |
| IP address type | IPv4 |
| VPC | `path-routing-vpc` |
| Mappings | `public-subnet-1`, `public-subnet-2` |
| Security group | `path-alb-sg` |
| Listener | HTTP 80 |
| Default action | Forward to `aws-tg` |

## 12. Add Listener Rules

Open `path-routing-alb`, go to **Listeners and rules**, select the HTTP listener, and add:

| Priority | Condition | Action |
| --- | --- | --- |
| `10` | `/aws` and `/aws/*` | Forward to `aws-tg` |
| `20` | `/azure` and `/azure/*` | Forward to `azure-tg` |
| `30` | `/gcp` and `/gcp/*` | Forward to `gcp-tg` |

## 13. Test

Copy the ALB DNS name.

Open:

```text
http://<alb-dns-name>/aws/
http://<alb-dns-name>/azure/
http://<alb-dns-name>/gcp/
```

Expected:

| URL | Result |
| --- | --- |
| `/aws/` | AWS backend page |
| `/azure/` | Azure backend page |
| `/gcp/` | GCP backend page |

## Important Note About the GitHub Repo

The original GitHub repo has richer HTML pages. Private instances cannot clone GitHub without NAT Gateway or another outbound path.

For this lab, the private instances use simple HTML from user data so the path-based routing concept works exactly like the document.
