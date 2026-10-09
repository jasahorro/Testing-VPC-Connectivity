<img src="https://cdn.prod.website-files.com/677c400686e724409a5a7409/6790ad949cf622dc8dcd9fe4_nextwork-logo-leather.svg" alt="NextWork" width="300" />

# Testing VPC Connectivity

**Project Link:** [View Project](https://nextwork.ai/projects/da091986-ef01-5973-a070-b15e174f59d8)

**Author:** Maria Jasmin Ahorro  
**Email:** jasminahorro23@gmail.com

---

![Image](https://nextwork.ai/overjoyed_magenta_proud_yak/uploads/da091986-ef01-5973-a070-b15e174f59d8_8ee57662)

## Introducing Today's Project!

### What is Amazon VPC?

Amazon VPC is an isolated virtual network dedicated to your AWS account, and it is useful because it allows you to define and control your own network environment—including IP address ranges, subnets, route tables, and network gateways—so you can securely host and isolate your cloud resources.

### How I used Amazon VPC in this project

In today's project, I used Amazon VPC to test network traffic routing and security controls by connecting to EC2 instances across public and private subnets, verifying inter-subnet communication with ping, and validating outbound internet access using curl.

### One thing I didn't expect in this project was...

One thing I didn't expect in this project was how multiple layers of network security—such as security groups, network ACLs, route tables, and internet gateways—all have to work together perfectly in order for traffic to flow between subnets and the internet.

### This project took me...

This project took me about 30 minutes to complete, including setting up the VPC infrastructure, configuring security groups and network ACLs, testing inter-instance connectivity with ping, and verifying outbound internet access using curl.

## Connecting to an EC2 Instance

Connectivity means how well different components and resources within your network talk to each other and communicate with external networks, allowing data to flow smoothly across systems.

My first connectivity test was whether I could connect to NextWork Public Server using EC2 Instance Connect, which initially failed due to missing SSH inbound rules in its security group.

![Image](https://nextwork.ai/overjoyed_magenta_proud_yak/uploads/da091986-ef01-5973-a070-b15e174f59d8_88727bef)

## EC2 Instance Connect

I connected to my EC2 instance using EC2 Instance Connect, which is a browser-based tool that provides direct SSH access to your EC2 instance through the AWS Management Console, automatically handling temporary SSH key pairs so you don't need to manually create or store SSH keys.

My first attempt at getting direct access to my public server resulted in an error, because the security group attached to NextWork Public Server only allowed inbound HTTP traffic, blocking the SSH traffic required by EC2 Instance Connect.

I fixed this error by editing the inbound rules of NextWork Public Security Group to add a new rule allowing SSH traffic on port 22 from Anywhere-IPv4 (0.0.0.0/0).

![Image](https://nextwork.ai/overjoyed_magenta_proud_yak/uploads/da091986-ef01-5973-a070-b15e174f59d8_1cbb1b88)

## Connectivity Between Servers

Ping is a core networking utility used to test whether a host or server can communicate with another device across a network by sending ICMP echo request packets and measuring the response time.

I used ping to test the connectivity between NextWork Public Server and NextWork Private Server by sending packets from the public instance to the private instance's private IPv4 address.

The ping command I ran was ping 10.0.1.189

The first ping returned "0 received, 100% packet loss" (or timed out without a reply). This meant that the private server's network firewall or security settings were blocking incoming ICMP echo requests from the public server, preventing successful communication between the two subnets.

![Image](https://nextwork.ai/overjoyed_magenta_proud_yak/uploads/da091986-ef01-5973-a070-b15e174f59d8_defghijk)

## Troubleshooting Connectivity

I troubleshooted this by updating both the private Network ACL (NACL) and the private security group to allow inbound and outbound ICMP - IPv4 traffic from the public subnet and security group, which enabled the private server to respond to ping requests from the public server.

![Image](https://nextwork.ai/overjoyed_magenta_proud_yak/uploads/da091986-ef01-5973-a070-b15e174f59d8_4a9e8014)

## Connectivity to the Internet

Curl is a command-line tool used for transferring data to or from a server using various protocols (like HTTP and HTTPS), which allows you to test network connectivity and retrieve web page content or raw HTML directly from your terminal.

I used curl to test the connectivity between my public EC2 instance and the public internet by sending HTTP requests to external domains like example.com and NextWork's web application to retrieve raw HTML content and verify proper internet gateway routing.

### Ping vs Curl

Ping and curl are different because ping operates at the network layer using ICMP packets to test basic reachability and measure round-trip latency between two hosts, whereas curl operates at the application layer using protocols like HTTP and HTTPS to transfer data and fetch the actual content or raw HTML of a web server.

## Connectivity to the Internet

I ran the curl command curl example.com which returned the raw HTML content of the target webpage, confirming that outbound internet access and internet gateway routing are functioning correctly.

![Image](https://nextwork.ai/overjoyed_magenta_proud_yak/uploads/da091986-ef01-5973-a070-b15e174f59d8_8ee57662)

---

*Built with [NextWork](https://nextwork.ai) - [View this project](https://nextwork.ai/projects/da091986-ef01-5973-a070-b15e174f59d8)*
