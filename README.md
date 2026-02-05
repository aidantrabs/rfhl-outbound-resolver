# Route 53 Outbound Resolver — Cross-Subnet DNS Resolution

## Current Setup
`TODO: validate this is accurate`

```
┌─────────────────────────────────────────────────────────┐
│  VPC (e.g. 10.0.0.0/16)                                 │
│                                                         │
│  Subnet 1 (e.g. 10.0.1.0/24)                            │
│  ┌──────────────┐                                       │
│  │ Windows EC2  │──── DNS query (port 53) ────┐         │
│  └──────────────┘                             │         │
│                                               ▼         │
│  Subnet 2 (e.g. 10.0.2.0/24)                            │
│  ┌───────────────────────────┐                          │
│  │ Outbound Resolver ENIs    │                          │
│  │  • ENI-A (10.0.2.x)       │                          │
│  │  • ENI-B (10.0.2.y)       │                          │
│  └───────────┬───────────────┘                          │
│              │                                          │
└──────────────┼──────────────────────────────────────────┘
               │  Resolver Forwarding Rule
               ▼
        ┌──────────────┐
        │ External DNS │  (2 target IPs in the rule)
        │  Servers     │
        └──────────────┘
```

## How It Works

1. EC2 sends a DNS query to the **VPC DNS resolver** (the `.2` address — e.g. `10.0.0.2`)
2. VPC DNS matches a **forwarding rule** for the domain
3. VPC DNS hands the query to the **Outbound Resolver ENIs** on Subnet 2
4. Outbound Resolver forwards the query to the **external DNS target IPs** defined in the rule
5. Response flows back the same path

> **Key point:** The EC2 instance does NOT talk directly to the Outbound Resolver ENIs. It talks to the VPC's built-in DNS (`10.0.0.2` / AmazonProvidedDNS), which then uses the Outbound Resolver under the hood.

---

## Configuration Checklist

### 1. EC2 DNS Settings (Subnet 1)

- [ ] **DHCP Options Set** on the VPC should use `AmazonProvidedDNS` (this is the default)
- [ ] On the Windows EC2, the DNS server should be the **VPC DNS resolver** (VPC CIDR base + 2, e.g. `10.0.0.2`)
  - Check: `ipconfig /all` — the DNS server should be `10.0.0.2`, **not** the outbound resolver ENI IPs
- [ ] **`enableDnsSupport`** is `true` on the VPC (Settings → Edit DNS settings)
- [ ] **`enableDnsHostnames`** is `true` on the VPC

### 2. Outbound Resolver Endpoint (Subnet 2)

- [ ] Outbound Resolver has **at least 2 ENIs** (likely already in place)
- [ ] ENIs are in **Subnet 2** with valid private IPs
- [ ] The **Security Group** on the Outbound Resolver ENIs allows:

| Direction  | Protocol | Port | Source/Dest              | Purpose                          |
|------------|----------|------|--------------------------|----------------------------------|
| **Inbound**  | TCP/UDP  | 53   | VPC CIDR (`10.0.0.0/16`) | Accept DNS from VPC DNS resolver |
| **Outbound** | TCP/UDP  | 53   | `0.0.0.0/0` (or target DNS IPs) | Forward to external DNS servers  |

### 3. Resolver Forwarding Rule

- [ ] Rule **type** is `FORWARD`
- [ ] Rule **domain** matches the domain being resolved (e.g. `corp.example.com` or `.` for all)
- [ ] Rule has the **2 external DNS server IPs** as targets (port 53)
- [ ] Rule is **associated with the VPC**

### 4. Network ACLs (Often the Gotcha)

**Subnet 1 NACLs** (where EC2 lives):

| Direction  | Protocol | Port       | Source/Dest    | Action |
|------------|----------|------------|----------------|--------|
| Outbound   | UDP/TCP  | 53         | VPC CIDR       | ALLOW  |
| Inbound    | UDP/TCP  | 1024-65535 | VPC CIDR       | ALLOW  |

**Subnet 2 NACLs** (where Resolver ENIs live):

| Direction  | Protocol | Port       | Source/Dest          | Action |
|------------|----------|------------|----------------------|--------|
| Inbound    | UDP/TCP  | 53         | VPC CIDR             | ALLOW  |
| Outbound   | UDP/TCP  | 53         | External DNS IPs     | ALLOW  |
| Inbound    | UDP/TCP  | 1024-65535 | External DNS IPs     | ALLOW  |
| Outbound   | UDP/TCP  | 1024-65535 | VPC CIDR             | ALLOW  |

> If using **default NACLs** (allow all), this won't be the issue. But if custom NACLs are in place, this is the most common cross-subnet blocker.

### 5. Route Tables

- [ ] Both subnets can route to each other (automatic within the same VPC via the `local` route)
- [ ] Subnet 2 has a route to the external DNS IPs (e.g. via NAT Gateway, Transit Gateway, VPN, or Direct Connect — depends on where those DNS servers live)

### 6. EC2 Security Group (Subnet 1)

| Direction  | Protocol | Port | Dest           | Purpose       |
|------------|----------|------|----------------|---------------|
| Outbound   | UDP/TCP  | 53   | VPC CIDR or `10.0.0.2/32` | DNS queries   |

> Most default SGs allow all outbound, so this is usually fine.

---

## Debugging

```powershell
# 1. Confirm DNS server on the Windows EC2
ipconfig /all
# Look for "DNS Servers" — should be 10.0.0.2 (VPC resolver)

# 2. Test DNS resolution
nslookup target.domain.com
nslookup target.domain.com 10.0.0.2

# 3. If nslookup fails, check if VPC DNS is reachable
Test-NetConnection -ComputerName 10.0.0.2 -Port 53

# 4. Check Route 53 Resolver query logs (if enabled)
# Route 53 → Resolver → Query Logging in AWS Console
```

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| EC2 DNS pointing directly at the Outbound Resolver ENI IPs | Set DNS to `10.0.0.2` (VPC resolver) — the VPC resolver handles forwarding |
| Resolver rule not associated with the VPC | Associate the rule with the VPC in the Route 53 console |
| Security group on resolver ENIs too restrictive | Allow inbound DNS (53) from VPC CIDR |
| Custom NACLs blocking cross-subnet traffic | Allow DNS + ephemeral ports between subnets |
| Outbound resolver can't reach external DNS | Ensure Subnet 2 has a route out (NAT GW, TGW, VPN, etc.) |
| Resolver rule domain doesn't match the query | Verify the domain in the rule covers what's being queried |

---

## TL;DR

The EC2 instance points at the VPC DNS (`10.0.0.2`), **not** the Outbound Resolver IPs directly. The forwarding rule needs to be associated with the VPC, and security groups + NACLs must allow DNS traffic. The VPC DNS automatically routes matching queries through the Outbound Resolver.
