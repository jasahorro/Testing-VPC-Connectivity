# Testing VPC Connectivity: Cheat Sheet

> 📚 **Part 5 of 9** in my [AWS VPC Series](https://github.com/jasahorro/AWS-VPC-Series) · ← Previous: [Launching VPC Resources](https://github.com/jasahorro/Launching-VPC-Resources/blob/main/CHEATSHEET.md) · Next: [VPC Peering](https://github.com/jasahorro/VPC-Peering/blob/main/CHEATSHEET.md) →

Quick-review notes for the NextWork **Testing VPC Connectivity** project.

## Contents

1. [The Big Picture](#1-the-big-picture)
2. [Key Terms](#2-key-terms)
3. [The Layers Traffic Must Pass](#3-the-layers-traffic-must-pass)
4. [Security Groups vs. Network ACLs](#4-security-groups-vs-network-acls)
5. [Test 1: EC2 Instance Connect (SSH)](#5-test-1-ec2-instance-connect-ssh)
6. [Test 2: Ping Between Servers (ICMP)](#6-test-2-ping-between-servers-icmp)
7. [Test 3: Curl to the Internet (HTTP)](#7-test-3-curl-to-the-internet-http)
8. [Ping vs. Curl](#8-ping-vs-curl)
9. [Troubleshooting Checklist](#9-troubleshooting-checklist)
10. [Commands Used](#10-commands-used)
11. [Self-Quiz](#11-self-quiz)

---

## 1. The Big Picture

Building a VPC is only half the job. You then **test** that traffic actually flows the way you designed it:

| Test | From → To | Tool | Protocol / Port | What it proves |
|---|---|---|---|---|
| 1 | You → Public Server | EC2 Instance Connect | SSH / TCP 22 | You can log in to your instance |
| 2 | Public Server → Private Server | `ping` | ICMP | The two subnets can talk to each other |
| 3 | Public Server → Internet | `curl` | HTTP / TCP 80 (HTTPS 443) | The internet gateway and routing work |

Each test **failed first**. Fixing each failure taught you which layer was blocking it.

---

## 2. Key Terms

| Term | What it is | Analogy |
|---|---|---|
| **VPC** | Your own isolated, private network in AWS | A gated neighborhood you own |
| **Subnet** | A section of the VPC's IP range, inside one Availability Zone | A street in the neighborhood |
| **Public subnet** | A subnet whose route table sends `0.0.0.0/0` to an internet gateway | A street connected to the highway |
| **Private subnet** | A subnet with no route to an internet gateway | A cul-de-sac with no highway exit |
| **Internet gateway (IGW)** | Connects the VPC to the internet | The highway on-ramp |
| **Route table** | Rules that decide where traffic goes next | Road signs |
| **Security group (SG)** | A firewall around an **instance** | The front-door lock on each house |
| **Network ACL (NACL)** | A firewall around a **subnet** | The security gate at the street entrance |
| **EC2 instance** | A virtual server | A house |
| **Private IPv4 address** | Address used inside the VPC (e.g. `10.0.1.189`) | A house number on the street |
| **Connectivity** | How well resources can communicate with each other and with outside networks | Whether the roads, gates and doors let you through |

---

## 3. The Layers Traffic Must Pass

For a packet to reach an instance, **every** layer must allow it:

```
Internet
   │
[ Internet Gateway ]   ← attached to the VPC?
   │
[ Route Table ]        ← is there a route (e.g. 0.0.0.0/0 → igw)?
   │
[ Network ACL ]        ← subnet level: inbound AND outbound allowed?
   │
[ Security Group ]     ← instance level: inbound allowed?
   │
 EC2 instance
```

> **Key lesson:** if any one layer blocks the traffic, the connection fails. When testing fails, check each layer in turn.

---

## 4. Security Groups vs. Network ACLs

This is the most important comparison in the project (and a common exam topic).

| | Security Group | Network ACL |
|---|---|---|
| **Protects** | An instance (its network interface) | A whole subnet |
| **Rule types** | **Allow only** | **Allow and Deny** |
| **State** | **Stateful**: return traffic is allowed automatically | **Stateless**: return traffic needs its own rule |
| **Rule order** | All rules are checked together | Checked in number order, lowest first; the first match wins |
| **Default (custom-created)** | Denies all inbound, allows all outbound | A new custom NACL **denies all** inbound and outbound |
| **Default (VPC's default)** | — | The default NACL allows all traffic |
| **Can reference** | Other security groups (e.g. "allow from Public SG") | CIDR ranges only |

**Stateful vs. stateless analogy:**
- **Security group** is a bouncer who remembers you. If you were let out, you're let back in.
- **NACL** is a bouncer with no memory. You have to show your ticket on the way in **and** on the way out.

That's why the ping fix needed **inbound and outbound** ICMP rules on the private NACL.

---

## 5. Test 1: EC2 Instance Connect (SSH)

**What it is:** a browser-based way to SSH into an instance from the AWS Console. AWS pushes a temporary SSH key for you, so you don't manage key pairs.

| | |
|---|---|
| **Protocol / port** | SSH, TCP **22** |
| **Symptom** | Connection failed |
| **Cause** | The Public Security Group only allowed **HTTP (80)** inbound, not SSH |
| **Fix** | Add an inbound rule: **SSH, TCP 22, source `0.0.0.0/0`** |

> **Security caveat:** `0.0.0.0/0` means anyone on the internet can try to connect on port 22. It's fine for a lab. In real use, restrict it to your own IP, or to the EC2 Instance Connect IP range for your region.

**Requirements for EC2 Instance Connect to work:**
- The instance has a **public IP** and sits in a **public subnet** (route to the IGW)
- The SG allows **inbound TCP 22** from the Instance Connect source
- The NACL allows inbound 22 **and** the outbound return traffic (ephemeral ports 1024–65535)
- An AMI with Instance Connect installed (Amazon Linux has it by default)

---

## 6. Test 2: Ping Between Servers (ICMP)

**What it is:** `ping` sends **ICMP echo requests** and waits for **echo replies**. It checks whether a host is reachable and how long the round trip takes (latency).

| | |
|---|---|
| **Command** | `ping 10.0.1.189` (the private server's private IP) |
| **Symptom** | `100% packet loss`, no replies |
| **Cause** | ICMP was blocked by the **private NACL** and the **private security group** |
| **Fix 1: Private NACL** | Allow **ICMP – IPv4** **inbound** from the public subnet CIDR **and outbound** back to it (stateless!) |
| **Fix 2: Private SG** | Allow inbound **ICMP – IPv4** with **source = Public Security Group** |

**Reading ping output:**

| Output | Meaning |
|---|---|
| `64 bytes from 10.0.1.189: icmp_seq=1 ttl=255 time=0.5 ms` | ✅ Reply received; time = latency |
| Nothing prints / hangs | ❌ Packets are being dropped (firewall, NACL, or route) |
| `100% packet loss` | ❌ No replies came back at all |

> **Tip:** ping runs forever on Linux. Press **Ctrl + C** to stop it and see the summary. Use `ping -c 4 <ip>` to send just 4 packets.

> **Why use the public server to ping?** The private server has no public IP, so you can't reach it from the internet. You test it from *inside* the VPC.

---

## 7. Test 3: Curl to the Internet (HTTP)

**What it is:** `curl` transfers data to or from a server over protocols like HTTP and HTTPS. In this project it fetched a web page's **raw HTML** to prove the instance can reach the internet.

| | |
|---|---|
| **Command** | `curl example.com` |
| **Success looks like** | Raw HTML printed in the terminal (`<!doctype html>...`) |
| **What it proves** | Outbound traffic works: SG outbound → NACL → route table → IGW → internet, and back |

**If curl fails, check:**
- Route table has `0.0.0.0/0 → igw-xxxx`
- The internet gateway is **attached** to the VPC
- The NACL allows outbound 80/443 **and** inbound return traffic on ephemeral ports (1024–65535)
- The instance has a **public IP**
- DNS resolution is enabled in the VPC (so `example.com` turns into an IP)

---

## 8. Ping vs. Curl

| | `ping` | `curl` |
|---|---|---|
| **Protocol** | ICMP | HTTP / HTTPS (over TCP) |
| **Network layer (OSI)** | Layer 3: Network | Layer 7: Application |
| **Question it answers** | "Can I reach this host at all?" | "Can I get actual content from this server?" |
| **Returns** | Echo replies + latency | The page content (HTML, JSON, etc.) |
| **Uses ports?** | No (ICMP has no ports) | Yes: 80 (HTTP), 443 (HTTPS) |
| **Analogy** | Knocking on the door to see if anyone's home | Asking them to hand you a letter |

> A host can block ping but still serve web pages, and the reverse is also possible. Use the tool that matches what you're testing.

---

## 9. Troubleshooting Checklist

When a connection fails, go through this list in order:

1. **Right target?** Correct IP (private IP from inside the VPC, public IP from outside)?
2. **Route table:** Does the subnet have a route to the destination (IGW for internet, `local` for inside the VPC)?
3. **Internet gateway:** Attached to the VPC? (only needed for internet traffic)
4. **NACL inbound:** Is the protocol/port allowed into the destination subnet?
5. **NACL outbound:** Is the **return** traffic allowed out? (stateless!)
6. **Security group inbound:** Does the destination instance allow this protocol/port from this source?
7. **Public IP:** Does the instance need one, and does it have one?

| Symptom | Most likely culprit |
|---|---|
| EC2 Instance Connect fails | SG missing inbound SSH (22), or no public IP |
| Ping between subnets times out | ICMP not allowed in SG and/or NACL (both directions) |
| curl hangs | Missing IGW route, NACL blocking return traffic, or no public IP |
| curl says "Could not resolve host" | DNS problem, not a firewall problem |

---

## 10. Commands Used

```bash
# Test reachability of the private server (Ctrl + C to stop)
ping 10.0.1.189

# Send only 4 pings, then stop
ping -c 4 10.0.1.189

# Fetch a web page's raw HTML
curl example.com

# Show only the response headers (quick check)
curl -I example.com
```

---

## 11. Self-Quiz

Try to answer before expanding.

<details>
<summary>1. Why did the first EC2 Instance Connect attempt fail?</summary>

The Public Security Group only allowed inbound HTTP (port 80). EC2 Instance Connect uses SSH on port 22, which wasn't allowed.
</details>

<details>
<summary>2. What rule fixed it?</summary>

An inbound SG rule: SSH, TCP 22, source `0.0.0.0/0` (Anywhere-IPv4).
</details>

<details>
<summary>3. What protocol does ping use, and does it have ports?</summary>

ICMP. ICMP has no ports, so you allow it by protocol type ("All ICMP – IPv4"), not by port.
</details>

<details>
<summary>4. Why did you need both inbound AND outbound ICMP rules on the NACL, but only inbound on the security group?</summary>

NACLs are stateless, so the reply needs its own outbound rule. Security groups are stateful, so the reply is allowed automatically.
</details>

<details>
<summary>5. Which acts first on traffic coming into an instance: the NACL or the security group?</summary>

The NACL, at the subnet boundary. The security group is checked next, at the instance.
</details>

<details>
<summary>6. Can a security group have a Deny rule?</summary>

No. Security groups only have Allow rules. Anything not allowed is denied. Use a NACL if you need an explicit Deny.
</details>

<details>
<summary>7. Why did you ping the private server's private IP, not a public one?</summary>

The private server has no public IP and no route to the internet. It's only reachable from inside the VPC.
</details>

<details>
<summary>8. What did a successful <code>curl example.com</code> prove?</summary>

That the public server has working outbound internet access: the route table, internet gateway, NACL and SG all allow the traffic and its return.
</details>

<details>
<summary>9. At which OSI layers do ping and curl work?</summary>

Ping: Layer 3 (Network). Curl: Layer 7 (Application).
</details>

<details>
<summary>10. In a NACL, if rule 100 allows ICMP and rule 90 denies all traffic, what happens to ICMP?</summary>

It's denied. NACL rules are checked from the lowest number up, and the first match wins, so rule 90 matches first.
</details>

<details>
<summary>11. Why is SSH from <code>0.0.0.0/0</code> risky in production?</summary>

It lets anyone on the internet attempt to connect. Restrict the source to your IP or to the EC2 Instance Connect IP range for your region.
</details>

<details>
<summary>12. curl returns "Could not resolve host". Is that a security group problem?</summary>

Probably not. It's a DNS problem: the name couldn't be turned into an IP. Check that DNS resolution is enabled in the VPC.
</details>
