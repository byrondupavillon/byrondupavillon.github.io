---
title: "Setting Up a Tailscale Subnet Router in AWS"
date: 2025-05-09
categories:
  - networking
tags:
  - tailscale
  - aws
  - subnet router
  - remote access
  - vpn
---

## Introduction

Tailscale is a user-friendly, secure mesh VPN built on WireGuard that simplifies connecting devices across various environments. One powerful feature of Tailscale is the **Subnet Router**, which allows you to route traffic from your Tailscale network to private subnets in cloud or on-prem environments.

In this guide, you’ll learn how to set up a Tailscale Subnet Router on an EC2 instance in AWS, enabling access to an entire VPC subnet from your Tailscale network.

---

## Prerequisites

Before starting, make sure you have:

- An AWS account with permissions to launch EC2 instances.
- A Tailscale account with admin access.
- Basic Linux command line knowledge.

---

## Step 1: Launch an EC2 Instance

1. Go to the AWS EC2 console.
2. Launch a new instance using a lightweight AMI (e.g., Amazon Linux 2 or Ubuntu Server).
3. Place the instance in the VPC and subnet you want to route traffic to.
4. Enable **Auto-assign Public IP** (or set up a bastion later).
5. Under security groups:
   - Allow inbound SSH (for setup).
   - Allow inbound traffic from `100.64.0.0/10` (Tailscale IP range) if you expect traffic from Tailscale nodes.

![Tailscale Subnet Router AWS Network Diagram](assets/img/posts/2025-05-09-tailscale-subnet-router-setup/aws-tailscale-setup.png)

---

## Step 2: Configure the Instance for IP Forwarding

SSH into the instance and enable IP forwarding:

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

### (Optional) Linux Optimizations for Subnet Routers and Exit Nodes

If your EC2 instance is running **Tailscale v1.54 or later** with **Linux kernel 6.2 or newer**, you can enable performance optimizations for UDP throughput using transport layer offloads. This is particularly useful for instances acting as **exit nodes** or **subnet routers**.

#### Enable Transport Layer Offloads (Runtime)

Use `ethtool` to configure the network interface:

```bash
NETDEV=$(ip -o route get 8.8.8.8 | cut -f 5 -d " ")
sudo ethtool -K $NETDEV rx-udp-gro-forwarding on rx-gro-list off
```

> 💡 These changes are **not persistent** and will be lost after a reboot.

---

#### Make Changes Persistent on Boot

If your distribution uses `networkd-dispatcher`, you can persist the settings with a boot-time script.

First, check if `networkd-dispatcher` is enabled:

```bash
systemctl is-enabled networkd-dispatcher
```

Then, create the script:

```bash
printf '#!/bin/sh\n\nethtool -K %s rx-udp-gro-forwarding on rx-gro-list off \n' "$(ip -o route get 8.8.8.8 | cut -f 5 -d " ")" | sudo tee /etc/networkd-dispatcher/routable.d/50-tailscale
sudo chmod 755 /etc/networkd-dispatcher/routable.d/50-tailscale
```

---

#### Test the Script

Run the following to verify it executes successfully:

```bash
sudo /etc/networkd-dispatcher/routable.d/50-tailscale
test $? -eq 0 || echo 'An error occurred.'
```

---

> 🧠 These optimizations can significantly improve performance for high-throughput Tailscale routing workloads. Only apply them on supported kernels and network interfaces.

---

## Step 3: Install and Authenticate Tailscale

Install Tailscale:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Authenticate and enable subnet routing:

```bash
sudo tailscale up \
    --hostname=tailscale-subnet-router-$date \
    --ssh \
    --advertise-connector \
    --advertise-routes=10.10.0.0/16 \
    --advertise-tags=tag:tailscale-subnet-router,tag:sysadmin-ssh-access \
    --authkey="${OAUTH_CLIENT_SECRET}?ephemeral=false&preauthorized=true"
```

### Explanation of `tailscale up` Options

Here’s a breakdown of the key flags used in the `tailscale up` command for configuring your subnet router:

- `--hostname=tailscale-subnet-router-$date`: Sets a unique, timestamped hostname for the Tailscale node to help with identification in the admin console.
- `--ssh`: Enables [Tailscale SSH](https://tailscale.com/kb/1088/tailscale-ssh/), allowing secure, keyless SSH access between Tailscale devices.
- `--advertise-connector`: Marks this node as a Tailscale Connector (used for funnel or ingress features). While optional, this is useful in more advanced networking scenarios.
- `--advertise-routes=10.10.0.0/16`: Makes this node a **subnet router**, advertising the specified CIDR range to your Tailscale network. Replace this with your actual VPC CIDR.
- `--advertise-tags=tag:tailscale-subnet-router,tag:sysadmin-ssh-access`: Assigns [ACL tags](https://tailscale.com/kb/1068/acl-tags/) to the node for fine-grained access control via Tailscale’s ACL policy configuration.
- `--authkey="${OAUTH_CLIENT_SECRET}?ephemeral=false&preauthorized=true"`: Authenticates the node using a pre-generated auth key. The `ephemeral=false` flag ensures the node persists in your network, and `preauthorized=true` skips admin approval during registration. Mainly used for headless setups or when you want to avoid manual approval.

> 💡 **Note**: Ensure your auth key is created with the appropriate scopes and TTL in the Tailscale admin console.

---

## Step 4: Approve Routes in Tailscale Admin Panel (If not using auth key)
If you used an auth key, skip this step. Otherwise, you need to approve the advertised routes in the Tailscale admin panel.

Log in to [Tailscale Admin Console](https://login.tailscale.com/admin/machines), find your EC2 instance, and **enable subnet routing** by approving the advertised route.

---

## Step 5: Enable Source/Destination Check Override (Important!)

By default, EC2 instances block forwarded traffic unless the **source/destination check** is disabled.

1. Go to EC2 Console.
2. Select your instance → Actions → Networking → Change source/dest. check.
3. **Disable** it.

![Source/Destination Check](assets/img/posts/2025-05-09-tailscale-subnet-router-setup/aws-source-dest-check.png)

---

## Step 6: Testing

From any device on your Tailscale network, try pinging a private IP in the VPC:

```bash
ping 10.0.1.10
```

You should now have access to AWS private subnet hosts via your secure Tailscale mesh.

---

## Optional: Running as a System Service

Enable Tailscale to start at boot:

```bash
sudo systemctl enable --now tailscaled
```

---

## Conclusion

Setting up a Tailscale Subnet Router in AWS enables seamless, secure access to your cloud infrastructure without complex VPNs or SSH tunnels. It's ideal for remote teams, debugging, and managing internal services.

---

## References

- [Tailscale Subnet Routers](https://tailscale.com/kb/1019/subnets/)
- [AWS Source/Dest Checks](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-network-security.html#source-dest-check)
