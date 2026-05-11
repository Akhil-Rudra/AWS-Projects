# Troubleshooting

## Why We Are Not Using Public Backend Servers

The document uses private backend servers, and this lab follows that design.

The Application Load Balancer is public. The EC2 web servers stay private.

## Why We Are Not Using NAT Gateway

KodeKloud Studio may not allow NAT Gateway.

Without NAT Gateway, private instances cannot:

- install packages from the internet
- clone GitHub repositories
- download Apache/httpd

So the workaround is to use built-in Python from Amazon Linux user data.

## Target Group Is Unhealthy

Check:

- The private instance is running.
- The private instance is registered on port 80.
- The health check path matches the page:
  - `/aws/`
  - `/azure/`
  - `/gcp/`
- `private-web-sg` allows HTTP 80 from `path-alb-sg`.
- The user data finished successfully.

## Check the Web Server From AWS Console Connect

If EC2 Instance Connect Endpoint is available, connect to the private instance and run:

```bash
ps aux | grep "python3 -m http.server"
curl http://localhost/aws/
curl http://localhost/azure/
curl http://localhost/gcp/
```

Only the matching path for that server should return the expected page.

## Restart the Simple Web Server

From the instance:

```bash
cd /var/www/html
sudo nohup python3 -m http.server 80 > /tmp/path-routing-web.log 2>&1 &
```

## Cannot Connect to Private EC2

Private instances do not have public IP addresses.

Use **EC2 Instance Connect Endpoint** if KodeKloud Studio allows it.

If it is not available, you can still complete the project because user data starts the web servers automatically.

## ALB Cannot Reach Private Servers

Check:

- ALB is in the two public subnets.
- Private instances are in private subnets.
- Target groups use the same VPC as the instances.
- `private-web-sg` allows HTTP 80 from `path-alb-sg`.
- ALB security group allows HTTP 80 from the internet.
