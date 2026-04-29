# AWS VPC Project: Public NGINX Proxy to Private App

This is a simple hands-on AWS project you can build from the AWS Console. The goal is to create a VPC with public and private subnets, place one server in each layer, and use NGINX on the public server as a proxy to reach the private server.

- One VPC
- Two public subnets across two Availability Zones
- Two private subnets across two Availability Zones
- Application Load Balancer for user traffic
- Internet Gateway for public internet access
- Public route table and private route table
- Security groups for controlled traffic
- One public EC2 instance running NGINX as a reverse proxy
- One private EC2 instance running a basic HTML page with Python's built-in web server

## Architecture

![AWS VPC architecture diagram](assets/architecture.svg)

### Resource Placement

| Layer | Resource | Location | Why |
| --- | --- | --- | --- |
| Public entry | Application Load Balancer | Public Subnet 1 and Public Subnet 2 | Users access the app through the ALB DNS name |
| Public server | NGINX proxy EC2 | Public Subnet 1 | Receives ALB traffic and forwards it privately |
| Private server | App EC2 | Private Subnet 1 | Runs the HTML/app page with no public IP |
| Internet access | Internet Gateway | Attached to VPC | Lets public resources reach the internet |

Important placement note:

- The **Public EC2 / NGINX proxy** must be in **Public Subnet 1** because it needs a public IPv4 address and a route to the Internet Gateway.
- The **Private EC2 / app server** must be in **Private Subnet 1** because it should not be reachable directly from the internet.
- The ALB lives in both public subnets and forwards traffic to the public EC2 proxy.

## Architecture Summary

The Application Load Balancer is the public entry point for end users. Users open the ALB DNS name in a browser, and the ALB forwards HTTP traffic to the public EC2 instance running NGINX.

The public EC2 instance runs NGINX as a reverse proxy and forwards browser traffic to the private EC2 instance. The private server has no public IP address, so users cannot reach it directly.

Because this was built in KodeKloud AWS Playground, the private app uses Python's built-in web server and the networking stays simple.

## Suggested Values

Use these values to keep the project simple and easy to explain:

| Resource | Value |
| --- | --- |
| Region | `us-east-1` or your preferred region |
| Availability Zone 1 | `us-east-1a` if using `us-east-1` |
| Availability Zone 2 | `us-east-1b` if using `us-east-1` |
| VPC CIDR | `10.0.0.0/16` |
| Public Subnet 1 | `10.0.1.0/24` |
| Public Subnet 2 | `10.0.2.0/24` |
| Private Subnet 1 | `10.0.11.0/24` |
| Private Subnet 2 | `10.0.12.0/24` |
| EC2 AMI | Amazon Linux 2023 |
| EC2 type | `t2.micro` or `t3.micro` |

## Step 1: Create the VPC

1. Open AWS Console.
2. Go to **VPC**.
3. Choose **Create VPC**.
4. Select **VPC only**.
5. Name it `simple-vpc-nginx-vpc`.
6. Set IPv4 CIDR to `10.0.0.0/16`.
7. Create the VPC.

## Step 2: Create Subnets

Create four subnets inside the VPC:

| Name | CIDR | Type | Availability Zone |
| --- | --- | --- | --- |
| `simple-vpc-public-1` | `10.0.1.0/24` | Public | `us-east-1a` |
| `simple-vpc-public-2` | `10.0.2.0/24` | Public | `us-east-1b` |
| `simple-vpc-private-1` | `10.0.11.0/24` | Private | `us-east-1a` |
| `simple-vpc-private-2` | `10.0.12.0/24` | Private | `us-east-1b` |

Note: if you choose a different region, use the first two Availability Zones shown in that region. For example, in another region the names may look like `us-west-2a` and `us-west-2b`. Keep this same pattern:

| Availability Zone | Subnets |
| --- | --- |
| AZ 1 | `simple-vpc-public-1` and `simple-vpc-private-1` |
| AZ 2 | `simple-vpc-public-2` and `simple-vpc-private-2` |

After creating the public subnets:

1. Select each public subnet.
2. Open **Actions > Edit subnet settings**.
3. Enable **Auto-assign public IPv4 address**.

Do not enable auto-assign public IPv4 address for the private subnets.

## Step 3: Create and Attach an Internet Gateway

1. Go to **Internet Gateways**.
2. Create an Internet Gateway named `simple-vpc-igw`.
3. Attach it to `simple-vpc-nginx-vpc`.

## Step 4: Create Route Tables

Create a public route table:

1. Name it `simple-vpc-public-rt`.
2. Associate it with both public subnets.
3. Add this route:

| Destination | Target |
| --- | --- |
| `0.0.0.0/0` | Internet Gateway |

This route makes the public subnets public. The ALB and public proxy EC2 depend on this route.

Create a private route table:

1. Name it `simple-vpc-private-rt`.
2. Associate it with both private subnets.
3. Keep only the default local VPC route.

The private route table should not have a `0.0.0.0/0` internet route in this project.

## Step 5: Create Security Groups

Create `alb-sg`:

| Type | Source | Purpose |
| --- | --- | --- |
| HTTP `80` | `0.0.0.0/0` | Let end users reach the load balancer |

Outbound can stay as the default **allow all**.

Create `public-proxy-sg`:

| Type | Source | Purpose |
| --- | --- | --- |
| HTTP `80` | `alb-sg` | Only the load balancer can reach the proxy |
| SSH `22` | Your IP only | Optional server access |

Outbound can stay as the default **allow all**.

When adding the HTTP rule, choose **Custom** as the source type and select the `alb-sg` security group as the source. This is better than allowing `0.0.0.0/0` because users should enter through the ALB.

Create `private-app-sg`:

| Type | Source | Purpose |
| --- | --- | --- |
| HTTP `80` | `public-proxy-sg` | Only the public proxy can reach the private app |
| SSH `22` | `public-proxy-sg` | Optional testing from the public server |

Outbound can stay as the default **allow all**.

When adding the HTTP rule, choose **Custom** as the source type and select `public-proxy-sg`. This keeps the private app reachable only from the proxy server.

## Step 6: Launch the Private App Server

Launch the private EC2 instance first. This server runs the actual app page, but it does not have a public IP address.

| Setting | Value |
| --- | --- |
| Name | `simple-vpc-private-app` |
| AMI | Amazon Linux 2023 |
| Instance type | `t2.micro` or `t3.micro` |
| Key pair | Create or choose a key pair, for example `simple-vpc-key` |
| Subnet | `simple-vpc-private-1` |
| Auto-assign public IP | Disabled |
| Security group | `private-app-sg` |

In **Network settings**, choose **Select existing security group** and select `private-app-sg`. Do not choose **Create security group** here.

After launch, copy the private IP address. You need it later for SSH testing and for the NGINX proxy configuration.

You cannot SSH directly from your laptop to this private EC2 instance because it has no public IPv4 address and no internet route. Use the public EC2 instance as a bastion host:

```text
Your laptop -> Public EC2 / bastion -> Private EC2
```

The private app security group must allow SSH `22` from `public-proxy-sg`. This keeps SSH access private and only allows the public proxy server to reach the private server.

Do not install NGINX on the private EC2. The private EC2 only needs Python's built-in HTTP server.

## Step 7: Launch the Public Proxy Server

Launch the public EC2 instance next. This server has a public IP address, acts as the SSH bastion host, and runs NGINX as a reverse proxy.

Use the same key pair that you selected for the private app server.

| Setting | Value |
| --- | --- |
| Name | `simple-vpc-public-proxy` |
| AMI | Amazon Linux 2023 |
| Instance type | `t2.micro` or `t3.micro` |
| Key pair | Same key pair, for example `simple-vpc-key` |
| Subnet | `simple-vpc-public-1` |
| Auto-assign public IP | Enabled |
| Security group | `public-proxy-sg` |

In **Network settings**, choose **Select existing security group** and select `public-proxy-sg`. Do not create a new launch-wizard security group.

After launch, copy the public IPv4 address of this public EC2 instance.

## Step 8: Connect to the Private Server Through the Public Server

On your local machine, make the key file private:

```bash
chmod 400 simple-vpc-key.pem
```

Add the key to your local SSH agent:

```bash
ssh-add simple-vpc-key.pem
```

Connect to the public EC2 with SSH agent forwarding:

```bash
ssh -A -i simple-vpc-key.pem ec2-user@PUBLIC_PROXY_PUBLIC_IP
```

From the public EC2, connect to the private EC2:

```bash
ssh ec2-user@PRIVATE_APP_PRIVATE_IP
```

This works because the public EC2 can reach the private EC2 inside the VPC, while your private key stays on your local machine.

## Step 9: Configure the Private App Server

Run these commands on the **private EC2**.

Create a simple HTML app:

```bash
sudo mkdir -p /opt/private-app

sudo tee /opt/private-app/index.html > /dev/null <<'HTML'
<!doctype html>
<html>
<head>
  <title>Private App Server</title>
</head>
<body>
  <h1>Hello from the private subnet</h1>
  <p>This page is served by the private EC2 instance.</p>
  <p>The private server has no public IP address.</p>
</body>
</html>
HTML
```

Create a systemd service for the Python web server:

```bash
sudo tee /etc/systemd/system/private-app.service > /dev/null <<'SERVICE'
[Unit]
Description=Private app HTTP server
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/private-app
ExecStart=/usr/bin/python3 -m http.server 80
Restart=always

[Install]
WantedBy=multi-user.target
SERVICE
```

Start the private app:

```bash
sudo systemctl daemon-reload
sudo systemctl enable private-app
sudo systemctl start private-app
sudo systemctl status private-app
```

Test it from the private EC2:

```bash
curl http://localhost
```

You should see the HTML page.

## Step 10: Configure NGINX on the Public Proxy Server

Exit back to the public EC2:

```bash
exit
```

Run these commands on the **public EC2**.

Install and start NGINX:

```bash
sudo dnf install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```

Replace the main NGINX config so the public EC2 becomes a reverse proxy.

Set `PRIVATE_APP_IP` to your private EC2 private IP address.

```bash
PRIVATE_APP_IP="PRIVATE_APP_PRIVATE_IP"

sudo tee /etc/nginx/nginx.conf > /dev/null <<NGINX
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log notice;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    access_log /var/log/nginx/access.log;

    server {
        listen 80;
        server_name _;

        location / {
            proxy_pass http://\${PRIVATE_APP_IP};
            proxy_set_header Host \$host;
            proxy_set_header X-Real-IP \$remote_addr;
            proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto \$scheme;
        }
    }
}
NGINX
```

Example:

```bash
PRIVATE_APP_IP="10.0.11.227"
```

Test and restart NGINX:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

Test from the public EC2:

```bash
curl http://PRIVATE_APP_PRIVATE_IP
curl http://localhost
```

Both commands should return the private app HTML page.

If `curl http://localhost` returns the NGINX welcome page, NGINX is still using its default config. If it returns `502 Bad Gateway`, the private app service is not running or port `80` is blocked between the public and private security groups.

## Step 11: Create the Target Group and Application Load Balancer

Create a target group:

| Setting | Value |
| --- | --- |
| Target type | Instances |
| Name | `simple-vpc-proxy-tg` |
| Protocol | HTTP |
| Port | `80` |
| VPC | `simple-vpc-nginx-vpc` |
| Health check path | `/` |

Do not choose **Application Load Balancer** as the target type here. That option is for chaining load balancers together. For this project, the ALB sends traffic to the public EC2 instance running NGINX, so the target type must be **Instances**.

Register only the `simple-vpc-public-proxy` EC2 instance as the target:

1. Select `simple-vpc-public-proxy`.
2. Keep the port as `80`.
3. Click **Include as pending below**.
4. Click **Register pending targets**.

Do not register the private EC2 instance in this target group. The target group should send traffic to the public NGINX proxy, and NGINX forwards the request to the private app.

Create an Application Load Balancer:

| Setting | Value |
| --- | --- |
| Name | `simple-vpc-app-alb` |
| Scheme | Internet-facing |
| IP address type | IPv4 |
| VPC | `simple-vpc-nginx-vpc` |
| Subnets | `simple-vpc-public-1` and `simple-vpc-public-2` |
| Security group | `alb-sg` |
| Listener | HTTP `80` |
| Forward to | `simple-vpc-proxy-tg` |

Wait until the target group shows the proxy instance as **healthy**.

If the target is unhealthy, check these first:

1. `public-proxy-sg` allows HTTP `80` from `alb-sg`.
2. NGINX is running on the public proxy EC2 instance.
3. The public proxy EC2 instance is registered in the target group on port `80`.
4. The public proxy subnet route table has `0.0.0.0/0` pointing to the Internet Gateway.
5. The target group health check path is `/`.

## Step 12: Test the Application

1. Copy the ALB DNS name.
2. Open it in your browser using `http://ALB_DNS_NAME`.
3. You should see the private app page.

That confirms:

1. Your browser reached the Application Load Balancer.
2. The ALB forwarded traffic to the public NGINX proxy.
3. The public EC2 instance used NGINX to proxy the request.
4. The private EC2 instance responded from the private subnet.

## How Traffic Flows

1. You open the ALB DNS name in your browser.
2. The Application Load Balancer receives the request.
3. The ALB forwards the request to the public EC2 NGINX proxy.
4. NGINX forwards the request to the private EC2 instance.
5. The private EC2 instance returns the HTML page.
6. The private EC2 instance stays private because it has no public IP and no internet route.

## Screenshot Checklist

Use these screenshots for your GitHub README:

- VPC details page
- Public and private subnet list
- Route tables showing public route to Internet Gateway
- Private route table showing local-only route
- Internet Gateway attached to the VPC
- Application Load Balancer listener and target group health
- Security groups inbound rules
- EC2 instances showing one public and one private instance
- Browser showing the ALB DNS name loading the private app page

## Captured Screenshots

<details>
<summary>VPC and Subnets</summary>

### VPC Created

![VPC list](screenshots/aws-console/02-vpc-list.png)

### VPC Resource Map

![VPC resource map](screenshots/aws-console/03-vpc-resource-map.png)

### Subnets Created

![Subnets list](screenshots/aws-console/04-subnets-list.png)

### Public Subnet 1

![Public subnet 1](screenshots/aws-console/05-public-subnet-1.png)

### Public Subnet 2

![Public subnet 2](screenshots/aws-console/06-public-subnet-2.png)

### Private Subnet 1

![Private subnet 1](screenshots/aws-console/07-private-subnet-1.png)

### Private Subnet 2

![Private subnet 2](screenshots/aws-console/08-private-subnet-2.png)

</details>

<details>
<summary>Internet Gateway and Routes</summary>

### Internet Gateway List

![Internet gateway list](screenshots/aws-console/09-internet-gateway-list.png)

### Internet Gateway Attached

![Internet gateway attached](screenshots/aws-console/10-internet-gateway-attached.png)

### Public Route Table

![Public route table](screenshots/aws-console/11-public-route-table.png)

### Public Route to Internet Gateway

![Public route to Internet Gateway](screenshots/aws-console/12-public-route-igw.png)

### Public Route Table Subnet Associations

![Public route table subnet associations](screenshots/aws-console/13-public-route-subnet-associations.png)

### Private Route Table Subnet Associations

![Private route table subnet associations](screenshots/aws-console/14-private-route-subnet-associations.png)

### Private Route Table Local Route

![Private route table local only](screenshots/aws-console/15-private-route-table-local-only.png)

</details>

<details>
<summary>Security Groups</summary>

### Create ALB Security Group

![Create ALB security group](screenshots/aws-console/16-create-alb-security-group.png)

### ALB Security Group Created

![ALB security group created](screenshots/aws-console/17-alb-security-group-created.png)

### Create Public Proxy Security Group

![Create public proxy security group](screenshots/aws-console/18-create-public-proxy-security-group.png)

### Public Proxy Security Group Created

![Public proxy security group created](screenshots/aws-console/19-public-proxy-security-group-created.png)

### Private App Security Group Created

![Private app security group created](screenshots/aws-console/20-private-app-security-group-created.png)

### Private App Security Group Rules

![Private app security group rules](screenshots/aws-console/21-private-app-security-group-rules.png)

</details>

<details>
<summary>EC2 Launch Configuration</summary>

### Private App Launch Network Settings

This screenshot shows why it is important to select the existing `private-app-sg` instead of creating a new security group during EC2 launch.

![Private app launch network warning](screenshots/aws-console/22-private-app-launch-network-warning.png)

### Public Proxy Launch Name

![Public proxy launch name](screenshots/aws-console/23-public-proxy-launch-name.png)

### Public Proxy Launch Network Settings

![Public proxy launch network settings](screenshots/aws-console/24-public-proxy-launch-network.png)

### Private App Instance Running

![Private app instance summary](screenshots/aws-console/25-private-app-instance-summary.png)

### Public Proxy Instance Running

![Public proxy instance summary](screenshots/aws-console/26-public-proxy-instance-summary.png)

### SSH to Public Proxy

![SSH to public proxy](screenshots/aws-console/27-ssh-to-public-proxy.png)

### NGINX Installed on Public Proxy

![NGINX install on public proxy](screenshots/aws-console/28-nginx-install-public-proxy.png)

### NGINX Active on Public Proxy

![NGINX active on public proxy](screenshots/aws-console/29-nginx-active-public-proxy.png)

### Private App Service Active

![Private app service active](screenshots/aws-console/30-private-app-service-active.png)

### Private App Localhost Test

![Private app localhost test](screenshots/aws-console/31-private-app-localhost-test.png)

### Public Proxy Reaches Private App

![Public proxy curl private app](screenshots/aws-console/32-public-proxy-curl-private-app.png)

</details>

<details>
<summary>Load Balancer and Target Group</summary>

### Target Group Settings

The target group uses **Instances** as the target type, because the ALB forwards traffic to the public EC2 NGINX proxy.

![Target group instances settings](screenshots/aws-console/33-target-group-instances-settings.png)

### Target Group Review

This review screen is useful, but make sure the public proxy instance is registered before relying on the ALB test. If it says no targets were added, go back to **Register targets** and add `simple-vpc-public-proxy` on port `80`.

![Target group review](screenshots/aws-console/34-target-group-review.png)

### ALB Listener Forwarding

The HTTP listener forwards traffic to `simple-vpc-proxy-tg`.

![ALB listener forward target group](screenshots/aws-console/35-alb-listener-forward-target-group.png)

### ALB Created

![ALB created](screenshots/aws-console/36-alb-created.png)

### ALB DNS Name

Use the ALB DNS name to test the application from your browser.

![ALB DNS name](screenshots/aws-console/37-alb-dns-name.png)

</details>

## Clean Up

Delete these resources when done:

- EC2 instances
- Application Load Balancer
- Target group
- Internet Gateway
- Subnets
- Route tables
- Security groups
- VPC

The Application Load Balancer and EC2 instances are the most important cost items to clean up quickly.
