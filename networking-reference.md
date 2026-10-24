# AWS VPC Peering Networking Reference Guide

This document provides a technical configuration reference for engineers building or auditing the dual-VPC peering architecture. It outlines the complete resource inventory, route table configurations, security group matrices, OpenSSH jump-host setups, and core AWS networking mechanics.

Related project documentation:
- [Architecture Specification](architecture.md)
- [Validation and Troubleshooting Guide](validation-and-troubleshooting.md)

---

## 1. Resource Inventory

Below is the complete inventory of all networking, compute, security, and administrative resources defined for this implementation.

| Logical Name / Tag | Resource Type | VPC / Parent | IPv4 CIDR / Addressing | Description & Purpose |
|---|---|---|---|---|
| `ewb-vpc-a` | `AWS::EC2::VPC` | AWS Account | `10.10.0.0/16` | Primary management and application VPC |
| `ewb-vpc-b` | `AWS::EC2::VPC` | AWS Account | `10.20.0.0/16` | Secondary isolated workload VPC |
| `ewb-a-public` | `AWS::EC2::Subnet` | `ewb-vpc-a` | `10.10.1.0/24` | Public subnet hosting the bastion host |
| `ewb-a-private` | `AWS::EC2::Subnet` | `ewb-vpc-a` | `10.10.2.0/24` | Private subnet hosting Application Server A |
| `ewb-b-private` | `AWS::EC2::Subnet` | `ewb-vpc-b` | `10.20.1.0/24` | Private subnet hosting Application Server B |
| `ewb-igw-a` | `AWS::EC2::InternetGateway` | `ewb-vpc-a` | N/A | Internet gateway attached to VPC A |
| `ewb-a-b-peer` | `AWS::EC2::VPCPeeringConnection` | VPC A & VPC B | N/A | Point-to-point VPC Peering link (`pcx-ID`) |
| `ewb-rt-a-public` | `AWS::EC2::RouteTable` | `ewb-vpc-a` | Associated: `ewb-a-public` | Public route table with default route to IGW |
| `ewb-rt-a-private` | `AWS::EC2::RouteTable` | `ewb-vpc-a` | Associated: `ewb-a-private` | Private route table with route to VPC B |
| `ewb-rt-b-private` | `AWS::EC2::RouteTable` | `ewb-vpc-b` | Associated: `ewb-b-private` | Private route table with return route to VPC A |
| `ewb-sg-bastion` | `AWS::EC2::SecurityGroup` | `ewb-vpc-a` | Bound to `ewb-bastion` | Ingress firewall for public SSH administration |
| `ewb-sg-app-a` | `AWS::EC2::SecurityGroup` | `ewb-vpc-a` | Bound to `ewb-app-a` | Ingress firewall for App-A SSH & ICMP |
| `ewb-sg-app-b` | `AWS::EC2::SecurityGroup` | `ewb-vpc-b` | Bound to `ewb-app-b` | Ingress firewall for App-B SSH & ICMP |
| `ewb-bastion` | `AWS::EC2::Instance` | `ewb-a-public` | Private: `10.10.1.10`<br/>Public: Dynamic IPv4 | Bastion jump host for operator administration |
| `ewb-app-a` | `AWS::EC2::Instance` | `ewb-a-private` | Private: `10.10.2.10`<br/>Public: None | Application instance in VPC A |
| `ewb-app-b` | `AWS::EC2::Instance` | `ewb-b-private` | Private: `10.20.1.10`<br/>Public: None | Application instance in VPC B |
| `ewb-lab` | `AWS::EC2::KeyPair` | Region-wide | RSA Private Key (`.pem`) | Key pair for OpenSSH client authentication |

> **Placeholder Reference**:
> - `pcx-ID`: The unique AWS ID assigned to the peering connection upon creation (e.g., `pcx-0a1b2c3d4e5f67890`).
> - `igw-ID`: The unique AWS ID assigned to the internet gateway upon creation (e.g., `igw-0123456789abcdef0`).

---

## 2. Route Table Reference

In AWS VPC networking, route tables govern subnet packet forwarding. Explicit subnet associations are mandatory to ensure subnets do not inadvertently inherit unmonitored default routes from the main route table.

### 2.1 Route Table: `ewb-rt-a-public`
- **Associated Subnet**: `ewb-a-public` (`10.10.1.0/24`)
- **Primary Function**: Routes outbound and inbound internet traffic for the bastion host and forwards administrative SSH connections targeted at VPC B across the peering connection.

| Destination CIDR | Target Gateway / Link | Route Status | Route Type | Purpose |
|---|---|---|---|---|
| `10.10.0.0/16` | `local` | Active | System (AWS-created) | Direct intra-VPC A communication (e.g., Bastion to App-A) |
| `0.0.0.0/0` | `igw-ID` (`ewb-igw-a`) | Active | User-configured | Default route for public internet access to the bastion |
| `10.20.0.0/16` | `pcx-ID` (`ewb-a-b-peer`) | Active | User-configured | Inter-VPC route enabling Bastion to reach App-B via SSH |

#### Longest Prefix Match (LPM) Explanation
When the bastion host (`10.10.1.10`) connects to `ewb-app-b` (`10.20.1.10`), the VPC router evaluates both `0.0.0.0/0` and `10.20.0.0/16`. Because `10.20.0.0/16` has a 16-bit prefix match versus a 0-bit prefix match for the default route, the router strictly forwards packets to `pcx-ID`. The presence of the internet gateway route never interferes with inter-VPC traffic.

---

### 2.2 Route Table: `ewb-rt-a-private`
- **Associated Subnet**: `ewb-a-private` (`10.10.2.0/24`)
- **Primary Function**: Isolates Application Server A from direct internet access while providing deterministic routing to VPC B across the peering connection.

| Destination CIDR | Target Gateway / Link | Route Status | Route Type | Purpose |
|---|---|---|---|---|
| `10.10.0.0/16` | `local` | Active | System (AWS-created) | Direct intra-VPC A communication (e.g., App-A to Bastion) |
| `10.20.0.0/16` | `pcx-ID` (`ewb-a-b-peer`) | Active | User-configured | Forwards App-A project traffic to VPC B (`10.20.0.0/16`) |

> Note: `ewb-rt-a-private` contains **no default route** (`0.0.0.0/0`). Subnet instances cannot send or receive internet traffic.

---

### 2.3 Route Table: `ewb-rt-b-private`
- **Associated Subnet**: `ewb-b-private` (`10.20.1.0/24`)
- **Primary Function**: Isolates Application Server B and provides the essential return path for all packets arriving from VPC A.

| Destination CIDR | Target Gateway / Link | Route Status | Route Type | Purpose |
|---|---|---|---|---|
| `10.20.0.0/16` | `local` | Active | System (AWS-created) | Direct intra-VPC B communication |
| `10.10.0.0/16` | `pcx-ID` (`ewb-a-b-peer`) | Active | User-configured | Return path for traffic destined for VPC A (`10.10.0.0/16`) |

#### The Requirement for Symmetric Bidirectional Routing
VPC Peering does not perform route injection or proxying. If `ewb-rt-a-private` has a route to `10.20.0.0/16`, forward packets successfully reach `10.20.1.10`. However, when `ewb-app-b` sends back response packets destined for `10.10.2.10`, the VPC B router searches `ewb-rt-b-private`. Without `10.10.0.0/16 -> pcx-ID`, the router drops the reply packets immediately. Forward and return paths must be configured on both sides.

---

## 3. Security Group Reference

AWS Security Groups function as virtual firewalls operating at the Elastic Network Interface (ENI) layer. All rules follow the principle of least privilege using host-level `/32` CIDR masks.

### 3.1 Security Group: `ewb-sg-bastion`
- **Parent VPC**: `ewb-vpc-a`
- **Associated Instance**: `ewb-bastion` (`10.10.1.10`)
- **Description**: Permits operator SSH access exclusively from the operator's current public IP address.

| Direction | Type | Protocol | Port Range | Source / Destination | Rationale & Purpose |
|---|---|---|---|---|---|
| Inbound | SSH | TCP | 22 | `<OPERATOR_PUBLIC_IP>/32` | Restricts management ingress to operator workstation |
| Outbound | All Traffic | All | All | `0.0.0.0/0` | Default egress rule for initiating SSH sessions |

> **Security Mandate**: Never configure the source for SSH as `0.0.0.0/0`. Exposing TCP port 22 to the public internet leaves the host vulnerable to automated brute-force attacks and port scans.

---

### 3.2 Security Group: `ewb-sg-app-a`
- **Parent VPC**: `ewb-vpc-a`
- **Associated Instance**: `ewb-app-a` (`10.10.2.10`)
- **Description**: Permits administrative SSH from the bastion host and ICMP Echo Requests from App-B.

| Direction | Type | Protocol | Port Range | Source / Destination | Rationale & Purpose |
|---|---|---|---|---|---|
| Inbound | SSH | TCP | 22 | `10.10.1.10/32` | Permits administrative access only from Bastion host |
| Inbound | Custom ICMP - IPv4 | ICMP | Type 8, Code 0 | `10.20.1.10/32` | Permits ping verification from App-B (`ewb-app-b`) |
| Outbound | All Traffic | All | All | `0.0.0.0/0` | Allows response packets and outbound ping requests |

---

### 3.3 Security Group: `ewb-sg-app-b`
- **Parent VPC**: `ewb-vpc-b`
- **Associated Instance**: `ewb-app-b` (`10.20.1.10`)
- **Description**: Permits administrative SSH from the bastion host and ICMP Echo Requests from App-A.

| Direction | Type | Protocol | Port Range | Source / Destination | Rationale & Purpose |
|---|---|---|---|---|---|
| Inbound | SSH | TCP | 22 | `10.10.1.10/32` | Permits administrative access only from Bastion host |
| Inbound | Custom ICMP - IPv4 | ICMP | Type 8, Code 0 | `10.10.2.10/32` | Permits ping verification from App-A (`ewb-app-a`) |
| Outbound | All Traffic | All | All | `0.0.0.0/0` | Allows response packets and outbound ping requests |

### 3.4 Key Security Distinctions
- **Port vs. Source Field**: The **Port** field specifies the listening service on the destination instance (e.g., TCP port 22). The **Source** field specifies the originating IP address or CIDR permitted to connect. Private IPs or CIDRs belong in the Source field.
- **ProxyJump IP Visibility**: When connecting to `ewb-app-b` through the bastion via OpenSSH `ProxyJump`, `ewb-bastion` opens a standard TCP socket from its private IP `10.10.1.10`. Therefore, `ewb-sg-app-b` must permit `10.10.1.10/32`, not the operator's laptop IP.

---

## 4. Private SSH Access Configuration

OpenSSH's native `ProxyJump` directive routes an administrative SSH session through an intermediary bastion host without requiring SSH agent forwarding, manual port forwarding, or copying private keys onto remote servers.

### 4.1 Client Configuration: `~/.ssh/config`

Create or append the following configuration block on the operator's local machine (`~/.ssh/config` on Linux, macOS, or WSL):

```sshconfig
# Bastion Host - Public Jump Host in VPC A
Host ewb-bastion
    HostName <BASTION_PUBLIC_IP>
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes

# Private Application Server A (VPC A)
Host ewb-app-a
    HostName 10.10.2.10
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes
    ProxyJump ewb-bastion

# Private Application Server B (VPC B via VPC Peering)
Host ewb-app-b
    HostName 10.20.1.10
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes
    ProxyJump ewb-bastion
```

### 4.2 Key File Permission Hardening

OpenSSH enforces strict permission checks on private key files. If permissions are too permissive, the client will abort authentication:

```bash
# Secure SSH directory and private key permissions on POSIX systems (Linux/macOS/WSL)
chmod 700 ~/.ssh
chmod 400 ~/.ssh/ewb-lab.pem
```

For native Windows PowerShell environments using Windows OpenSSH:
```powershell
# Reset inherited permissions and grant explicit read permission to current user only
icacls "$env:USERPROFILE\.ssh\ewb-lab.pem" /inheritance:r
icacls "$env:USERPROFILE\.ssh\ewb-lab.pem" /grant:r "$($env:USERNAME):(R)"
```

### 4.3 Operational Security Rules
1. **Never Commit Private Keys**: Never add `.pem` or private key files to Git repositories. Include `*.pem` and `id_*` in `.gitignore`.
2. **Never Copy Private Keys to EC2**: Never upload, paste, or transfer `ewb-lab.pem` to `ewb-bastion` or application instances. With `ProxyJump`, the private key remains exclusively on the operator's local computer.

---

## 5. Technical Notes on AWS Networking Mechanics

### 5.1 Stateful Security Groups
AWS security groups maintain a connection tracking state table:
- When an inbound packet matches a permitted rule, the security group automatically creates a temporary state table entry.
- All response packets belonging to that established session are automatically allowed outbound, regardless of outbound security group rules.
- Conversely, outbound requests automatically permit incoming responses.

### 5.2 Internet Gateway (IGW) Mechanics
An Internet Gateway is a horizontally scaled, redundant VPC component that performs two distinct roles:
1. **Routing Target**: Acts as the next-hop target in route tables for internet-bound CIDRs (`0.0.0.0/0`).
2. **One-to-One NAT**: Translates between the instance's private IPv4 address (`10.10.1.10`) and its public IPv4 address. The EC2 instance operating system has no knowledge of its public IP; it only sees its assigned private IP (`10.10.1.10`).

### 5.3 VPC Peering Infrastructure Characteristics
- **AWS SDN Routing**: Peering does not route traffic through a physical appliance, gateway instance, or VPN concentrator. It uses the underlying AWS network virtualization layer (Hyperplane / Andromeda).
- **Zero Bottlenecks**: Bandwidth between peered instances is identical to bandwidth between instances within the same VPC.
- **MTU Support**: Inter-VPC traffic within the same AWS Region supports standard 1500-byte MTU as well as 9001-byte Jumbo frames (for instances supporting jumbo frames).

### 5.4 Edge-to-Edge Limitation: Why VPC B Has No Internet
AWS explicitly prohibits transitive edge routing across VPC Peering connections.
- An Internet Gateway attached to VPC A can only service traffic originating from within VPC A's own IP address space.
- Traffic entering VPC A via peering from VPC B (`10.20.0.0/16`) cannot be forwarded out to `ewb-igw-a`.
- This isolation is enforced by the AWS virtualization plane and cannot be bypassed via route table modifications.
