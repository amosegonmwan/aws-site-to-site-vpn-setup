# AWS Site-to-Site VPN Setup

This repository provides a step-by-step guide to configure a Site-to-Site VPN connection between an on-premises environment and an AWS VPC. The setup involves creating a Virtual Private Gateway (VGW), a Customer Gateway (CGW), and establishing a VPN connection using OpenSwan. EC2 instances in different VPCs and regions are used to test connectivity.

## Project Overview

The objective of this project is to establish a secure communication channel between an on-premises data center and AWS using a Site-to-Site VPN connection.

## Components

- **Virtual Private Gateway (VGW):** Created on the AWS side. The VGW is the VPN concentrator on the Amazon side of the VPN connection.
- **Customer Gateway (CGW):** Represents the customer data center's VPN device. This resource specifies information about your on-premises device, such as the public IP address and the type of routing to be used.

In summary, the Virtual Private Gateway is created and managed on the AWS side, while the Customer Gateway represents your on-premises device and is also created as a resource in AWS to facilitate the VPN connection configuration.

## Prerequisites

- VPCs in different AWS regions (West and East).
- Basic understanding of AWS networking and VPN concepts.

![image](https://github.com/user-attachments/assets/01c2faf7-1e83-414e-a4f1-1d633bd8286c)

## Steps

### Step 1: Launch EC2 Instance in On-Premises VPC (West Region - Prod Account)

1. **Launch EC2 Instance:**
   - Choose an Amazon Machine Image (AMI).
   - Select an instance type (e.g., t2.micro).
   - Configure the instance to be within the public subnet of your VPC.
   - Disable source/destination checks.
   - Attach a security group allowing SSH (port 22) and ICMP (all).
   - Make a note of the public IP address.

### Step 2: AWS VPC Configuration (East Region - Management Account)

1. **Launch EC2 Instance:**
   - Choose an Amazon Machine Image (AMI).
   - Select an instance type (e.g., t2.micro).
   - Configure the instance to be within the public or private subnet of your VPC.
   - Attach a security group allowing SSH (port 22) and ICMP (all).

2. **Create Customer Gateway (CGW):**
   - Navigate to the VPC Dashboard in the AWS Management Console.
   - Select "Customer Gateways" from the left-hand menu.
   - Click "Create Customer Gateway".
   - Name: `CGW`
   - Routing: `static`
   - IP: Public IP of on-premises EC2 instance.

3. **Create Virtual Private Gateway (VGW):**
   - In the VPC Dashboard, select "Virtual Private Gateways" from the left-hand menu.
   - Click "Create Virtual Private Gateway".
   - Name: `VGW`
   - Attach to VPC.

4. **Create VPN Connection:**
   - In the VPC Dashboard, select "VPN Connections" from the left-hand menu.
   - Click "Create VPN Connection".
   - Name: `VPN`
   - Target type: `Virtual Private Gateway`
   - Select the CGW and VGW created earlier.
   - Routing: `static`
   - Enter prefixes: 172.31.0.0/16, 10.0.0.0/16 (VPC CIDRs for both East and West).
   - Download VPN configuration as `OpenSwan` type.

### Step 3: Enable Route Propagation for AWS VPC Route Table (East Region - Management Account)

1. **Enable Route Propagation:**
   - Navigate to the Route Tables section within the VPC Dashboard.
   - Select the route table associated with your VPC.
   - Edit route propagation and enable it for the VGW.

### Step 4: Configure OpenSwan on On-Premises VPC EC2 Instance (Production Account)

1. **Install OpenSwan:**
   ```sh
   sudo su
   yum install openswan -y
   ```

2. **Update sysctl Configuration:**
   ```sh
   nano /etc/sysctl.conf
   ```
   Add the following lines:
   ```sh
   net.ipv4.ip_forward = 1
   net.ipv4.conf.all.accept_redirects = 0
   net.ipv4.conf.all.send_redirects = 0
   ```
   Apply the changes:
   ```sh
   sysctl -p
   ```
   
3. **Configure OpenSwan Tunnels:**
   - Edit the OpenSwan configuration file:
     ```sh
     nano /etc/ipsec.d/aws.conf
     ```
   - Paste the tunnel configuration for TUNNEL 1 from the downloaded file. Example:
   ```sh
conn Tunnel1
  	authby=secret
  	auto=start
  	left=%defaultroute
  	leftid=3.145.65.10
  	right=52.11.172.243
  	type=tunnel
  	ikelifetime=8h
  	keylife=1h
  	phase2alg=aes128-sha1;modp1024
  	ike=aes128-sha1;modp1024
  	auth=esp
  	keyingtries=%forever
  	keyexchange=ike
  	leftsubnet=<LOCAL NETWORK>
  	rightsubnet=<REMOTE NETWORK>
  	dpddelay=10
  	dpdtimeout=30
  	dpdaction=restart_by_peer
   ```
4. **Configure OpenSwan Secrets:**
   - Edit the secrets file:
     ```sh
     nano /etc/ipsec.d/aws.secrets
     ```
   - Add the following line (replace with values from the downloaded config file):
   ```sh
   <Public IP of CGW> <Public IP of VGW>: PSK "<Pre-shared key>"
  
   E.g.,
  
   54.169.159.173 54.66.224.114: PSK "Vkm1hzbkdxLHb7wO2TJJnRLTdWH_n6u3"
   ```
5. **Start OpenSwan:**
   ```sh
   systemctl start ipsec
   systemctl status ipsec
   ```

### Step 5: Test Connectivity

1. **Test Connectivity:**
    - From the on-premises EC2 instance, test connectivity by pinging the private IP address of the EC2 instance in the AWS VPC:
      ```
      ping <private-ip-of-ec2-in-aws-vpc>
      ```
2. **Update Route Table:**
    - Ensure the route table in the on-premises network points to the OpenSwan instance for routing traffic to the AWS VPC.


**REMARK**
Alternatively, If you use Amazon Linux 2023,

This is an alternative for **`Openswan`** since its not supported in Amazon Linux 2023
```sudo vi /etc/yum.repos.d/fedora.repo``` 

```sh
[fedora]
name=Fedora 36 - $basearch
#baseurl=http://download.example/pub/fedora/linux/releases/36/Everything/$basearch/os/
metalink=https://mirrors.fedoraproject.org/metalink?repo=fedora-36&arch=$basearch
enabled=0
countme=1
metadata_expire=7d
repo_gpgcheck=0
type=rpm
gpgcheck=1
gpgkey=https://getfedora.org/static/fedora.gpg
skip_if_unavailable=False
```



