# How to Create and Tear Down a VPC and EC2 Instance

This guide provides both **overview (quick)** and **detailed (step-by-step)** instructions for creating an AWS VPC with subnets and an EC2 instance, and then safely tearing them down to avoid unnecessary charges.

---

## Part 1: Create a VPC and EC2 Instance

### Overview (Quick Guide)

1. **Plan in Google Sheets**
   - VPC CIDR: `10.10.0.0/16`
   - Public Subnets: `10.10.1.0/24`, `10.10.2.0/24`, `10.10.3.0/24`
   - Private Subnets: `10.10.11.0/24`, `10.10.12.0/24`, `10.10.13.0/24`
   - Spread across 3 Availability Zones

2. **Create VPC in AWS**
   - Go to **VPC → Create VPC and more**
   - Name: `Project1-VPC`
   - CIDR block: `10.10.0.0/16`
   - 3 public and 3 private subnets

3. **Networking Options**
   - Internet Gateway is created for public subnets
   - Optional: NAT Gateway for private subnets (extra cost)

4. **Security Groups**
   - Public SG: Allow ICMP, HTTP, SSH
   - Private SG: Allow ICMP (demo)

5. **Launch EC2**
   - Amazon Linux 2 AMI
   - Place in Project1-VPC, public subnet, with auto-assign public IP
   - Attach Public SG
   - Add user data script for web server
   - Launch and test by entering the public IP in your browser

---

### Detailed (Step-by-Step)

1. Open Google Sheets/Excel and create a diagram of the VPC:  
   - VPC CIDR: `10.10.0.0/16`  
   - Public: `10.10.1.0/24`, `10.10.2.0/24`, `10.10.3.0/24`  
   - Private: `10.10.11.0/24`, `10.10.12.0/24`, `10.10.13.0/24`  

2. Go to the AWS Console → search for **VPC** → click **Create VPC and more**.  

3. Enter:  
   - Name: `Project1-VPC`  
   - IPv4 CIDR block: `10.10.0.0/16`  
   - Number of Availability Zones: 3  
   - Number of Public Subnets: 3  
   - Number of Private Subnets: 3  

4. Leave NAT Gateway unchecked unless required (adds cost).  

5. Review and click **Create VPC**.  

6. Verify that the VPC and subnets were created successfully in the console.  

7. Create Security Groups:  
   - **SG-Public**  
     - Inbound: ICMP (ping), HTTP (80), SSH (22)  
     - Outbound: leave default  
   - **SG-Private**  
     - Inbound: ICMP (for demo)  
     - Outbound: leave default  

8. Launch an EC2 Instance:  
   - Name: `Project1-EC2`  
   - AMI: Amazon Linux 2  
   - Key Pair: optional  
   - Network: select `Project1-VPC`  
   - Subnet: choose one of the **public subnets**  
   - Auto-assign Public IP: Enable  
   - Security Group: choose **SG-Public**  

9. Under Advanced Settings → User Data, paste this script:  

   ```bash
   #!/bin/bash
   # Use this for your user data (script from top to bottom)
   # install httpd (Linux 2 version)
   yum update -y
   yum install -y httpd
   systemctl start httpd
   systemctl enable httpd

   # Get the IMDSv2 token
   TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

   # Background the curl requests
   curl -H "X-aws-ec2-metadata-token: $TOKEN" -s http://169.254.169.254/latest/meta-data/local-ipv4 &> /tmp/local_ipv4 &
   curl -H "X-aws-ec2-metadata-token: $TOKEN" -s http://169.254.169.254/latest/meta-data/placement/availability-zone &> /tmp/az &
   curl -H "X-aws-ec2-metadata-token: $TOKEN" -s http://169.254.169.254/latest/meta-data/network/interfaces/macs/ &> /tmp/macid &
   wait

   macid=$(cat /tmp/macid)
   local_ipv4=$(cat /tmp/local_ipv4)
   az=$(cat /tmp/az)
   vpc=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" -s http://169.254.169.254/latest/meta-data/network/interfaces/macs/${macid}/vpc-id)

   echo "
   <!doctype html>
   <html lang=\"en\" class=\"h-100\">
   <head>
   <title>Details for EC2 instance</title>
   </head>
   <body>
   <div>
   <h1>AWS Instance Details</h1>
   <h1>Samurai Katana</h1>

   <br>
   # insert an image or GIF
   <img src=\"https://www.w3schools.com/images/w3schools_green.jpg\" alt=\"W3Schools.com\">
   <br>

   <p><b>Instance Name:</b> $(hostname -f) </p>
   <p><b>Instance Private Ip Address: </b> ${local_ipv4}</p>
   <p><b>Availability Zone: </b> ${az}</p>
   <p><b>Virtual Private Cloud (VPC):</b> ${vpc}</p>
   </div>
   </body>
   </html>
   " > /var/www/html/index.html

   # Clean up the temp files
   rm -f /tmp/local_ipv4 /tmp/az /tmp/macid
Review settings and launch instance.

Wait until the instance state is running.

Copy the public IP address from the EC2 console.

Open a browser and go to http://<your-public-ip>.

Confirm that the custom metadata web page is displayed with your details and image.
