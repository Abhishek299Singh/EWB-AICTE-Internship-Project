# AWS VPC Peering Validation and Troubleshooting Guide

This guide establishes the verification procedures, acceptance criteria, diagnostic workflows, and decommissioning protocols for the AWS dual-VPC peering architecture.

Related project documentation:
- [Architecture Specification](architecture.md)
- [Networking Reference](networking-reference.md)

---

## 1. Prerequisites and Preflight Checks

Before performing functional verification tests, complete the following preflight audit to confirm that all infrastructure dependencies are properly configured.

| Component | Target State / Value | Verification Method |
|---|---|---|
| **VPC Peering Status** | `Active` (`pcx-ID`) | AWS Console > VPC > Peering Connections; confirm requester (`ewb-vpc-a`) and accepter (`ewb-vpc-b`) are accepted. |
| **Route Table Associations** | Explicit associations verified | Confirm `ewb-a-public` -> `ewb-rt-a-public`, `ewb-a-private` -> `ewb-rt-a-private`, `ewb-b-private` -> `ewb-rt-b-private`. |
| **Bidirectional Peering Routes** | `Active` status on target `pcx-ID` | Confirm `10.20.0.0/16` exists on both VPC A tables; confirm `10.10.0.0/16` exists on `ewb-rt-b-private`. |
| **Internet Gateway Route** | `0.0.0.0/0 -> ewb-igw-a` | Confirm default route is present **only** on `ewb-rt-a-public`. |
| **Security Group Ingress** | `/32` Host Scoping | Confirm rules match intended sources (`<OPERATOR_IP>/32` on bastion, `10.10.1.10/32` on apps for SSH, and peer `/32` for ICMP). |
| **Compute Instance States** | `running` / `2/2 checks passed` | Verify `ewb-bastion`, `ewb-app-a`, and `ewb-app-b` are operational in EC2 console. |
| **Private IP Allocations** | Static private IPs confirmed | Confirm `10.10.1.10` (Bastion), `10.10.2.10` (App-A), and `10.20.1.10` (App-B). |
| **Public IPv4 Isolation** | No public IPv4 on workloads | Verify in EC2 console that `Public IPv4 address` is empty (`-`) for both `ewb-app-a` and `ewb-app-b`. |

---

## 2. Validation Test Matrix

The following test matrix documents the mandatory verification steps required to validate private inter-VPC reachability, security group policy enforcement, and workload isolation.

> **Execution Status Notice**:
> In accordance with rigorous engineering documentation standards, the **Actual Result / Status** column reflects the pre-deployment verification baseline (`Pending verification`). Expected outcomes represent formal acceptance criteria and must not be misconstrued as completed test runs until executed within an active AWS lab account.

| Test ID & Name | Execution Location | Command or Action | Expected Technical Result (Acceptance Criteria) | Actual Result / Status |
|---|---|---|---|---|
| **TEST-01**<br/>Pre-Peering Baseline Isolation | `ewb-app-a`<br/>(via Bastion) | `hostname && hostname -I`<br/>`ping -c 4 10.20.1.10` | 4 packets transmitted, 0 received, 100% packet loss. Confirms total network isolation prior to peering route configuration. | Pending verification |
| **TEST-02**<br/>Forward Inter-VPC ICMP Reachability | `ewb-app-a`<br/>(after peering & routes) | `hostname && hostname -I`<br/>`ping -c 4 10.20.1.10` | 4 packets transmitted, 4 received, 0% packet loss. Round-trip time (RTT) reported. Confirms route to VPC B and App-B ICMP ingress acceptance. | Pending verification |
| **TEST-03**<br/>Reverse Inter-VPC ICMP Reachability | `ewb-app-b`<br/>(via Bastion ProxyJump) | `hostname && hostname -I`<br/>`ping -c 4 10.10.2.10` | 4 packets transmitted, 4 received, 0% packet loss. Confirms return routing symmetry from VPC B to VPC A and App-A ICMP ingress acceptance. | Pending verification |
| **TEST-04**<br/>Cross-Peering Administrative SSH | Operator Laptop | `ssh ewb-app-b` | Command successfully establishes encrypted session through `ewb-bastion` via OpenSSH `ProxyJump`. Shell opens as `[ec2-user@ip-10-20-1-10 ~]$`. | Pending verification |
| **TEST-05**<br/>Application Public IP Isolation Audit | Operator Laptop<br/>(AWS CLI / Console) | `aws ec2 describe-instances --filters "Name=tag:Name,Values=ewb-app-a,ewb-app-b" --query "Reservations[*].Instances[*].[Tags[?Key=='Name'].Value|[0],PublicIpAddress]"` | Output returns `null` or empty for both instances. Host metadata query `curl -s http://169.254.169.254/latest/meta-data/public-ipv4` returns HTTP 404. | Pending verification |
| **TEST-06**<br/>Controlled Security Group Failure & Recovery | `ewb-app-a` & AWS Console | 1. Remove ICMP rule from `ewb-sg-app-b`.<br/>2. Run `ping -c 4 10.20.1.10` from App-A.<br/>3. Re-add ICMP rule (`10.10.2.10/32`).<br/>4. Rerun `ping -c 4 10.20.1.10`. | Step 2 yields 100% packet loss (traffic drops immediately while SSH remains operational). Step 4 yields 0% packet loss (immediate recovery). Proves security group state enforcement. | Pending verification |

### Linux Host Inspection Snippet
To capture execution evidence during lab evaluation, run the following command block inside each private instance shell:

```bash
# Print hostname, assigned private IPv4, and test peer reachability
echo "=== Host Identification ==="
hostname
hostname -I

echo "=== Peer Reachability Test ==="
# Replace PEER_PRIVATE_IP with 10.20.1.10 (from App-A) or 10.10.2.10 (from App-B)
ping -c 4 <PEER_PRIVATE_IP>
```

---

## 3. Comprehensive Troubleshooting Guide

When connectivity fails, diagnose components systematically in the following order: **Instance State -> Host IP Configuration -> Subnet Route Tables -> Peering Status -> Security Groups -> Network ACLs**.

| Symptom | Probable Root Cause | Diagnostic Check | Resolution Procedure |
|---|---|---|---|
| **Bastion SSH times out**<br/>`ssh: connect to host BASTION_PUBLIC_IP port 22: Connection timed out` | 1. Operator public IP changed.<br/>2. Subnet route table missing IGW default route.<br/>3. Subnet not associated with `ewb-rt-a-public`. | 1. Run `curl -s https://checkip.amazonaws.com` on laptop.<br/>2. Check `ewb-sg-bastion` inbound rules.<br/>3. Check `ewb-rt-a-public` routes for `0.0.0.0/0 -> ewb-igw-a`. | Update `ewb-sg-bastion` to match current public IP `/32`. Ensure `ewb-a-public` is explicitly associated with `ewb-rt-a-public` and contains the `0.0.0.0/0` target. |
| **App-A SSH fails**<br/>Connection to `10.10.2.10:22` times out or is refused | 1. `ewb-sg-app-a` missing rule for Bastion.<br/>2. Source IP set incorrectly.<br/>3. Instance stopped. | 1. Verify `ewb-sg-app-a` permits TCP 22 from `10.10.1.10/32`.<br/>2. Verify instance private IP is `10.10.2.10`.<br/>3. Check instance state in EC2 console. | Add inbound rule: Type `SSH`, Port `22`, Source `10.10.1.10/32`. Verify `10.10.1.10` is entered in Source, not Port. |
| **App-B SSH fails via ProxyJump**<br/>`ssh: connect to host 10.20.1.10 port 22: Connection timed out` | 1. Public route table missing route to VPC B.<br/>2. Private route table in VPC B missing return route.<br/>3. Security group source misconfigured. | 1. Check `ewb-rt-a-public` for `10.20.0.0/16 -> pcx-ID`.<br/>2. Check `ewb-rt-b-private` for `10.10.0.0/16 -> pcx-ID`.<br/>3. Check `ewb-sg-app-b` allows TCP 22 from `10.10.1.10/32`. | Add missing route `10.20.0.0/16 -> pcx-ID` to `ewb-rt-a-public`. Add return route `10.10.0.0/16 -> pcx-ID` to `ewb-rt-b-private`. Update `ewb-sg-app-b` inbound rules. |
| **Ping fails after peering acceptance**<br/>`ping -c 4 10.20.1.10` yields 100% loss | 1. Peering connection in `Pending-acceptance` state.<br/>2. Inbound ICMP rule missing on target.<br/>3. Target IP address mismatch. | 1. Check Peering Connection status in AWS Console.<br/>2. Check `ewb-sg-app-b` for Custom ICMP IPv4 (Type 8, Code 0).<br/>3. Verify actual IP of `ewb-app-b`. | Accept peering request under Actions > Accept Request. Add ICMP Echo Request rule from `10.10.2.10/32`. Ensure instances use exact assigned IPs. |
| **Missing return route blackhole**<br/>Packets reach App-B but replies never return to App-A | Forward route exists in VPC A, but `ewb-rt-b-private` lacks route to VPC A | Run `tcpdump -nn -i any icmp` on App-B. If request packets arrive but no replies leave successfully, routing back is broken. | Add route to `ewb-rt-b-private`: Destination `10.10.0.0/16`, Target `pcx-ID` (`ewb-a-b-peer`). |
| **Incorrect security group source field**<br/>Rule saved but traffic blocked | Private IP was entered in Port Range or CIDR entered without `/32` | Inspect rule details in AWS Console: verify Port is `22` and Source is `10.10.1.10/32`. | Recreate security group rule ensuring Port is `22` and Source specifies the host CIDR mask. |
| **Peering creation rejected by AWS**<br/>`Cannot create peering connection between overlapping CIDR blocks` | Both VPCs were created with identical or intersecting CIDR blocks | Review VPC CIDRs in VPC console (`Your VPCs`). | Recreate one of the VPCs with non-overlapping address space. Use `10.10.0.0/16` for VPC A and `10.20.0.0/16` for VPC B. |
| **SSH Permission denied (publickey)**<br/>`ec2-user@10.20.1.10: Permission denied (publickey,gssapi-keyex,gssapi-with-mic)` | 1. Key file permissions too permissive.<br/>2. Wrong key used during instance launch.<br/>3. Incorrect username. | 1. Check local key permissions (`ls -l ~/.ssh/ewb-lab.pem`).<br/>2. Verify instance was launched with key pair `ewb-lab`.<br/>3. Verify SSH username is `ec2-user`. | Run `chmod 400 ~/.ssh/ewb-lab.pem`. If wrong key was selected at launch, relaunch instance with key pair `ewb-lab`. Note: Routing is functional if SSH authentication prompts. |
| **Stateless Network ACL blocking traffic**<br/>Custom NACL dropping return packets | Custom NACL applied without ephemeral port return rules | Check subnet Network ACL associations. Check inbound and outbound rules on both NACLs. | This architecture relies on default AWS NACLs (Allow All). If custom NACLs are required, inbound and outbound ephemeral port ranges (`1024-65535`) must be opened bidirectionally. |

---

## 4. Resource Teardown and Cleanup Checklist

To prevent ongoing cloud charges (e.g., EBS volume storage, public IPv4 address hourly pricing), execute the following teardown sequence in exact order upon completing testing and evidence capture.

> **Caution Before Teardown**:
> - Verify that all required verification screenshots, terminal outputs, and reports have been preserved locally. Terminated cloud resources cannot be recovered.
> - Ensure you delete **only** resources created for this project (`Project = EWB-VPC-Peering`). Do not delete default VPCs or production assets.

### Step 1: Compute and Credential Teardown
1. **Terminate EC2 Instances**:
   - In the EC2 console, select `ewb-bastion`, `ewb-app-a`, and `ewb-app-b`.
   - Choose **Instance state > Terminate instance**.
2. **Verify EBS Storage Deletion**:
   - Navigate to **Elastic Block Store > Volumes**.
   - Verify that all root volumes attached to the lab instances are deleted automatically (enabled via `Delete on Termination`). Manually delete any detached volumes.
3. **Release Elastic IP Addresses**:
   - Navigate to **Network & Security > Elastic IPs**.
   - If an Elastic IP was manually allocated for the bastion, select it and choose **Actions > Release Elastic IP address**.
4. **Remove Lab Key Pair**:
   - In **Network & Security > Key Pairs**, delete `ewb-lab`.
   - On the local workstation, remove the local file `~/.ssh/ewb-lab.pem` if no longer needed.

### Step 2: VPC Peering and Network Dependency Teardown
1. **Wait for ENIs to Clear**:
   - Wait 1–2 minutes until terminated instance network interfaces disappear from **Network Interfaces**.
2. **Delete Peering Connection**:
   - Navigate to **VPC > Peering Connections**.
   - Select `ewb-a-b-peer` and choose **Actions > Delete VPC peering connection**.
   - Confirm deletion. (Deleting the peering connection automatically marks associated peering routes as `blackhole`/inactive).
3. **Delete Custom Security Groups**:
   - In **VPC > Security Groups**, select `ewb-sg-app-a`, `ewb-sg-app-b`, and `ewb-sg-bastion`.
   - Choose **Actions > Delete security groups**.
4. **Remove Subnet Associations & Delete Route Tables**:
   - In **VPC > Route Tables**, select `ewb-rt-a-public`, `ewb-rt-a-private`, and `ewb-rt-b-private`.
   - Under **Subnet associations**, remove custom associations if necessary, then delete the custom route tables.
5. **Delete Subnets**:
   - In **VPC > Subnets**, select `ewb-a-public`, `ewb-a-private`, and `ewb-b-private`.
   - Choose **Actions > Delete subnet**.
6. **Detach and Delete Internet Gateway**:
   - In **VPC > Internet Gateways**, select `ewb-igw-a`.
   - Choose **Actions > Detach from VPC** (select `ewb-vpc-a`), then choose **Actions > Delete internet gateway**.
7. **Delete VPCs**:
   - In **VPC > Your VPCs**, select `ewb-vpc-a` and choose **Actions > Delete VPC**.
   - Select `ewb-vpc-b` and choose **Actions > Delete VPC**.

### Step 3: Account Verification and Cost Audit
1. **Tag Filter Check**:
   - Filter the AWS Management Console across EC2 and VPC services using tag `Project = EWB-VPC-Peering`. Confirm zero matching resources remain.
2. **Billing Console Review**:
   - Navigate to the **AWS Billing and Cost Management** console.
   - Check the **Bills** page over the next 24 hours to confirm that charges for EC2 compute, EBS volumes, and public IPv4 addresses have ceased. Note that AWS billing metric aggregation can lag by several hours.
