## peering-vpc
Connecting two isolated AWS VPCs privately using VPC peering, built in the AWS Console.

# Overview:-
This project creates two separate VPCs, a "dev" network and a "prod" network, and connects them with a VPC peering connection. An EC2 instance in the dev VPC can then reach an EC2 instance in the prod VPC using only private IP addresses, without the traffic ever crossing the public internet.
The prod instance has no public IP and no Internet Gateway, so the only way to reach it is through the peering connection.

# Architecture

Architecture diagram:-

        dev VPC (10.0.0.0/16)                     prod VPC (10.1.0.0/16)
   +---------------------------+              +---------------------------+
   |  dev subnet 10.0.1.0/24   |              |  prod subnet 10.1.1.0/24  |
   |   instance_dev            |   peering    |   instance_prod           |
   |   10.0.1.78 (public IP)   |<------------>|   10.1.1.42 (private)     |
   +---------------------------+              +---------------------------+
              |
      Internet Gateway (SSH access only)

# Network Plan:-

Item    	                         Dev                              	Prod
Region	                  ap-southeast-2 (Sydney)	          ap-southeast-2 (Sydney)
VPC name	                     dev_ishi_vpc	                   prod_ishi_vpc
VPC CIDR	                       10.0.0.0/16	                    10.1.0.0/16
Subnet	                     dev_subnet, 10.0.1.0/24	        prod_subnet, 10.1.1.0/24
Availability Zone	              ap-southeast-2a	                 ap-southeast-2a
Instance	                  instance_dev (t3.micro)           	instance_prod (t3.micro)
Private IP	                     10.0.1.78	                        10.1.1.42
Public IP	                    Yes (for SSH only)	                    None
Security group	                    dev_sg	                        prod_sg

# Resources Created:-
2-VPCs
2-subnets (one per VPC, same Availability Zone)
1-Internet Gateway (attached to the dev VPC only, used for SSH access)
1-VPC peering connection
2-route tables (the VPCs' main route tables, edited)
2-security groups
2-EC2 instances (t3.micro, Amazon Linux 2023)

## How I Built It:-
1.)Created the two VPCs
Used "VPC only" with manual IPv4 CIDR input. The ranges must not overlap, which is why dev uses 10.0.0.0/16 and prod uses 10.1.0.0/16.

2.)Created one subnet in each VPC
Both subnets are in the same Availability Zone to keep the test simple. Auto-assign public IPv4 was turned on for the dev subnet only.

3.)Attached an Internet Gateway to the dev VPC
Added a route 0.0.0.0/0 to the Internet Gateway in the dev route table, so I could SSH into the dev instance. The prod VPC has no Internet Gateway.

4.)Created and accepted the peering connection
Requester: dev VPC. Accepter: prod VPC. The status must change to Active after accepting the request.

5.)Added routes on both sides
Creating the peering connection does not create any routes. Both route tables needed an entry:-
Route table	Destination	Target
Dev	10.1.0.0/16	Peering connection
Prod	10.0.0.0/16	Peering connection

6.)Configured security groups
Security group	Rule	Source
dev_sg	SSH (22)	My IP only
dev_sg	All ICMP-IPv4	10.1.0.0/16 (prod VPC)
prod_sg	All ICMP-IPv4	10.0.0.0/16 (dev VPC)

7.)Launched the EC2 instances
Two t3.micro instances running Amazon Linux 2023: instance_dev in the dev subnet with a public IP, and instance_prod in the prod subnet with no public IP.

# Test Result:-
I connected to instance_dev over SSH and pinged the private IP of instance_prod:-
ping -c 4 10.1.1.42
Result: 4 packets transmitted, 4 received, 0% packet loss, with an average round-trip time of about 0.24 ms.
Ping success
This confirms that the two VPCs can communicate privately through the peering connection.

# What I Learned:-
CIDR ranges of peered VPCs must not overlap, otherwise the peering cannot be created.
Routing is manual. A peering connection does nothing until routes are added in both VPCs.
Peering is not transitive. If A is peered with B and B with C, A still cannot reach C without its own peering or a Transit Gateway.
Security groups still apply. An open network path does not mean traffic is allowed; the ICMP rules were needed on both sides.
A private instance (no public IP, no Internet Gateway) can still be reached from a peered VPC.

# Problems I Ran Into
| Region selector hidden in the console | Read the region from the URL (`ap-southeast-2`) |
| `ssh` not recognized in PowerShell | Installed OpenSSH Client with `Add-WindowsCapability` |
| `ssh` still not found after installing | Ran it by full path: `C:\Windows\System32\OpenSSH\ssh.exe` |

# Cost and Cleanup:-
Only the EC2 instances cost money, so after testing I terminated them and deleted the peering connection, Internet Gateway, subnets, and VPCs.

# Next Steps
- Add a third VPC to show peering is not transitive
