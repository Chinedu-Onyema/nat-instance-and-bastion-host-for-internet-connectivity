# AWS NAT Instance & Bastion Host Configuration Guide

This is a complete step-by-step repository guide on configuring a custom NAT Instance and Bastion Host on AWS to allow private subnet instances to access the internet securely without exposing them to inbound public access.

This design patterns a dual-purpose NAT Instance and Bastion Host inside a Public Subnet. 
The NAT instance routes outbound internet traffic for sensitive compute workloads living inside a Private Subnet while performing Network Address Translation (NAT) via iptables. 
Additionally, secure administrative access is enabled via SSH Agent Forwarding without storing private SSH keys on the intermediate server.

### PDF GUIDE: [NAT INSTANCE AND BASTION HOST CONFIGURATION FOR PRIVATE SUBNET RESOURCES.pdf](https://github.com/user-attachments/files/32192741/NAT.INSTANCE.AND.BASTION.HOST.CONFIGURATION.FOR.PRIVATE.SUBNET.RESOURCES.pdf)



### WATCH VIDEO WALKTHROUGH HERE: https://youtu.be/UEvKNXq1fvc



## PREREQUISITES

An active AWS Account with administrative access.  

Access to the target region (e.g., eu-north-1).  

A target VPC containing at least one Public Subnet and one Private Subnet (e.g., private CIDR 172.31.48.0/25).  

AWS CloudShell or a local CLI terminal with an SSH client.


## STEP-BY-STEP IMPLEMENTATION

### Phase 1: Create NAT Instance Security Group (NAT_SG)

1) Open the EC2 Console in eu-north-1 -> Security Groups -> Create Security Group.
      Name: NAT_SG | Description: Security group for NAT Instance and Bastion Host.  

2) Add Inbound Rules:HTTP (80): Source = Custom (172.31.48.0/25 - Private Subnet CIDR)  HTTPS (443):
      Source = Custom (172.31.48.0/25 - Private Subnet CIDR)  SSH (22): Source = Anywhere (0.0.0.0/0)

4) Add Inbound Rules:HTTP (80):
   Source = Custom (172.31.48.0/25 - Private Subnet CIDR)  HTTPS (443):
   Source = Custom (172.31.48.0/25 - Private Subnet CIDR)  SSH (22):
   Source = Anywhere (0.0.0.0/0)

5) Add Outbound Rules:HTTP (80):
   Destination = Anywhere (0.0.0.0/0)
   HTTPS (443): Destination = Anywhere (0.0.0.0/0)



### Phase 2: Launch & Configure the NAT Instance

1) Launch an EC2 Instance:
   Name: nat_instance
   
   AMI: Amazon Linux 2023
   
   Instance Type: t3.micro
   
   Key Pair: Create or select nat_keypair (.pem)
   
   Network Settings: Choose target VPC, a Public Subnet, auto-assign Public IP enabled, and attach NAT_SG.

3) SSH into nat_instance using CloudShell or your terminal.

3) Install, configure, and persist IP Tables NAT routing:


#### Install and enable iptables-services
<PRE>sudo yum install iptables-services -y</PRE>
<PRE>sudo systemctl enable iptables</PRE>
<PRE>sudo systemctl start iptables</PRE>

#### Enable IPv4 Packet Forwarding across reboots
<PRE>sudo nano /etc/sysctl.d/custom-ip-forwarding.conf</PRE>

  Add the following line to the file:
  <PRE>net.ipv4.ip_forward = 1</PRE>
  
  Apply sysctl parameters:
```bash
sudo sysctl -p /etc/sysctl.d/custom-ip-forwarding.conf
```


4) Identify Primary Network Interface & Configure Masquerading:

<PRE>netstat -i</PRE>

(Confirm interface name, e.g., ens5).


#### Configure NAT IP translation rule (replace ens5 if interface differs)
<PRE>sudo /sbin/iptables -t nat -A POSTROUTING -o ens5 -j MASQUERADE</PRE>
<PRE>sudo /sbin/iptables -F FORWARD</PRE>
<PRE>sudo service iptables save</PRE>



### Phase 3: Disable Source/Destination Checks

By default, EC2 instances enforce source/destination checks, preventing them from forwarding traffic intended for other devices.

1) Select nat_instance in the EC2 Console.

2) Click Actions -> Networking -> Change source/destination check.

3) Check Stop (Disable) and click Save



### Phase 4: Update Private Subnet Route Table

1) Open VPC Console -> Route Tables.

2) Select the route table attached to your private subnet (e.g., RDS-Pvt-rt).

3) Go to Routes tab -> Edit routes -> Add route:

   Destination: 0.0.0.0/0

   Target: Select Instance -> choose nat_instance (e.g., i-0e66c97394c1e5ca0)

4) Click Save changes



### Phase 5: Update NAT_SG for Bastion Host Capabilities

1) To enable SSH Bastion connectivity and diagnostics between subnets, update NAT_SG: 

   Inbound Rules -> Add Rule:Custom ICMP - IPv4: Source = Custom (172.31.48.0/25 - Private Subnet CIDR)  

   Outbound Rules -> Add Rules:SSH (22): Destination = Custom (172.31.48.0/25 - Private Subnet CIDR)  Custom ICMP - IPv4: Destination = Anywhere (0.0.0.0/0) 



### Phase 6: Launch Private Instance (PRIVATE_SG)

1) Launch an EC2 Instance:

   Name: private_instance

   AMI: Amazon Linux 2023  Instance Type: t3.micro

   Key Pair: Use nat_keypair

   Network Settings: Select target VPC, a Private Subnet (RDS-Pvt-subnet-1), Auto-assign Public IP = Disabled.



2) Create Security Group: PRIVATE_SG:

   Inbound SSH (22): Source = Custom (172.31.32.0/20 - Public Subnet CIDR hosting NAT Instance).



3) After launching, edit PRIVATE_SG Outbound Rules -> Add Rule:

   Custom ICMP - IPv4: Destination = Anywhere (0.0.0.0/0).



### Phase 7: End-to-End Testing via SSH Agent Forwarding
SSH Agent Forwarding securely passes your local SSH keys to remote hosts without saving private key files onto public bastions.

1) Start local SSH Agent & Load Key:

<PRE>eval $(ssh-agent)</PRE>
<PRE>ssh-add nat_keypair.pem</PRE>


2) Connect to NAT Instance using Agent Forwarding (-A)

<PRE>ssh -A ec2-user@<NAT_INSTANCE_PUBLIC_IP></PRE>


3) Verify Internet on NAT Instance:

<PRE>ping www.amazon.com</PRE>


4) SSH into Private Instance from NAT Instance:

<PRE>ssh ec2-user@<PRIVATE_INSTANCE_PRIVATE_IP></PRE>


5) Verify Outbound Internet Access from Private Instance:

<PRE>ping www.amazon.com</PRE>

   Expected Output: 0% packet loss. 
   Traffic flows from Private Instance ➔ NAT Instance ➔ Internet Gateway ➔ Target.


6) Exit instances and terminate SSH Agent:

<PRE>exit</PRE>
<PRE>exit</PRE>                 
<PRE>eval $(ssh-agent -k)</PRE> 



## TROUBLESHOOTING CHECKLIST
If outbound ping or package installation fails from the private instance, verify:

1) Source/Destination Check: Confirm that Source/Destination checking is Disabled on nat_instance.

2) Route Table Attachment: Verify 0.0.0.0/0 points directly to the nat_instance ID in the private subnet route table.

3) Security Group Rules: Ensure NAT_SG allows inbound HTTP, HTTPS, and ICMP traffic from the private subnet CIDR.

4) IP Forwarding & Masquerade: Ensure net.ipv4.ip_forward = 1 and iptables rules were saved successfully.

