# AWS Security Groups — Logic, Significance, Examples & Architectures

An AWS Security Group (SG) is a **stateful virtual firewall attached to supported AWS resources/network interfaces**. It controls which traffic is allowed **into** and **out of** a resource. Security groups use **allow rules only**; unlike Network ACLs, they don't have explicit deny rules. ([AWS Documentation][1])

![Image](https://images.openai.com/static-rsc-4/FnTDoQdgwylCoz0BBeJdOhYbJYp1_AqbSxkaMpN35S6hFFGPp7bq5hjavPRuTuEqOGOalxWGwOjQKAtJnm_5qvqttGMxmyQnVhg-eRYnc3LvIP2mBv3cNhufFZz6dyIZpWbjpN5Xi5-crOrEefhtZzXWzXBeRxXwHiQPN7JsnO6HTthskpP0qiTfnG_vNE0s?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/KyP_KphsMNT2cCsPYygjZfYkBoj2kMv68FUP3Di0X1U7JMVVTVStiw55zg2z8MbXgUt1qpQIN9gi9IiXr-AgcwLTwmokM__887YS6dgHFFiSxMlW2wmlRTkonbTj331XLxStF6SdbngUgz4t5QzQdy8F8qqcYK7GbKOu59W6s6KIC4hG6SSrPurVVPbxzTOm?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/mEkADVb4ib91WtF6qhwmib8sQWWmYZjZQriirRUThffqj-MhuiAiA0sIfXz6X_XFovu-7RHfFU6yKSs2Ap4DjIzJ_6ii0IJdxikYXT2G6IrTUgN6YsqvSIneEv8VkTrxKLgZG6DSId8t0huyQcSdvCIZRqJfER4D31MhqXLu9f8e4wBAzQeR7F_sJMLPnluN?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/2xDlo3EzibqMLDpMvWL3Il1Wl_YLhQ95RhvCofT_zWSleEZyFVsglGxGJVEFyFRxCRXcnhAWjGTLW2oY1ttFe6uTxq2Jsggfy6yQ7bigzQwGIRue6htGyCSYsU8WETNvk0kcmZYPK1nxOmZiMizfaRxhLpVRQIAVY5FrLwXD3JwG2xukJOBYT-AEbMXVDqQB?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/pXz8rUM2BE2XxQ7hJpCxuyDKCjgNg6AX2NVv0nrrLrITM-T_ZPU-gdAmG2n0SCJtaZlEPEYcHJZtlIMaiBck-pTsqz7ytblTLTo8Uk7TC8Uex1JGiOINeClZGHUx64iGS07XzPzwEn2d7E0xbY4bq8vkJdNzKof21_pyAkjAK4nOBXZ8duq_34ebfHDzrejB?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/0rJVG_Y_MEIG8DMskh_ldOxLDQuD3NOEyfW33C7RKTw_i_Q-ufC3qzKgtycrCYTOgTrcSZ9j7ZcX2SUIVHmhPfnTgnb9Bcc-Ivj6XNr0WHUb5rZs5xjj7zMM4TAgbXVwbMAgNAxVguNO6l6KYMEveKWZiDnQdlilV-X2K7X6XcJHGAfsPe0cfCSUgQbX1M-g?purpose=fullsize)

## 1. Core Security Group Logic

Think of the flow as:

```text
SOURCE
   │
   ▼
┌─────────────────────┐
│   Security Group    │
│                     │
│ Inbound Rule Check  │
└─────────┬───────────┘
          │ Allowed?
     ┌────┴────┐
     │         │
    NO        YES
     │         │
     X         ▼
            Resource
           EC2 / RDS
               │
               │ Response
               ▼
        Automatically Allowed
        because SG is STATEFUL
```

AWS security groups perform connection tracking. If an inbound connection is permitted, its response is permitted outbound even if an outbound rule would not independently permit that response, and vice versa. ([AWS Documentation][2])

The fundamental rule is:

```text
Inbound
SOURCE ────────> RESOURCE

Outbound
RESOURCE ──────> DESTINATION
```

For example:

```text
Inbound:
TCP 22    Source 203.0.113.10/32

means

203.0.113.10
      │
      │ SSH :22
      ▼
     EC2
```

It does **not** mean port 22 is open to everyone.

---

# 2. What Is Inside an SG Rule?

A typical rule contains:

```text
Protocol + Port + Source/Destination
```

Examples:

| Type       | Protocol |         Port | Source            | Meaning                                  |
| ---------- | -------- | -----------: | ----------------- | ---------------------------------------- |
| SSH        | TCP      |           22 | `203.0.113.10/32` | SSH from one IPv4 address                |
| HTTP       | TCP      |           80 | `0.0.0.0/0`       | HTTP from all IPv4 addresses             |
| HTTPS      | TCP      |          443 | `0.0.0.0/0`       | HTTPS from all IPv4 addresses            |
| MySQL      | TCP      |         3306 | `sg-app`          | MySQL from resources belonging to app SG |
| PostgreSQL | TCP      |         5432 | `sg-app`          | PostgreSQL from app SG                   |
| ICMP       | ICMP     | Echo Request | specific CIDR     | Ping                                     |

AWS permits sources/destinations such as IPv4/IPv6 CIDRs, prefix lists, and security-group IDs. ([AWS Documentation][1])

---

# 3. Example — Public Web Server

Suppose you have:

```text
                  INTERNET
                     │
             ┌───────┴───────┐
             │               │
           HTTP            HTTPS
            :80             :443
             │               │
             └───────┬───────┘
                     ▼
              ┌─────────────┐
              │    EC2      │
              │ Web Server  │
              └─────────────┘
```

Security Group:

```text
Inbound
---------------------------------
HTTP     TCP 80    0.0.0.0/0
HTTPS    TCP 443   0.0.0.0/0
SSH      TCP 22    YOUR-IP/32

Outbound
---------------------------------
All Traffic         0.0.0.0/0
```

AWS documents HTTP/HTTPS from `0.0.0.0/0` as a standard public-web-server pattern. ([AWS Documentation][3])

The important distinction is that web traffic can be public while administrative SSH remains restricted.

---

# 4. `/32` vs `/24` vs `/0`

CIDR size controls how many addresses potentially match.

```text
203.0.113.25/32
        │
        └── ONE IPv4 address

203.0.113.0/24
        │
        └── Network range

0.0.0.0/0
        │
        └── ALL IPv4 addresses
```

So:

```text
SSH 22 → 0.0.0.0/0
```

is highly exposed.

Whereas:

```text
SSH 22 → 203.0.113.25/32
```

limits SSH to that address.

AWS recommends avoiding unrestricted access to administrative ports such as SSH and RDP. ([AWS Documentation][4])

---

# 5. Security Group Referencing

One of the most significant SG features is that another **security group can be the source**.

Instead of:

```text
TCP 3306
Source = 10.0.2.15/32
```

you can configure:

```text
TCP 3306
Source = sg-app
```

Architecture:

```text
┌──────────────────────┐
│ Application EC2      │
│ SG = sg-app          │
└──────────┬───────────┘
           │
           │ TCP 3306
           ▼
┌──────────────────────┐
│ MySQL / RDS          │
│ SG = sg-db           │
│                      │
│ Inbound:             │
│ 3306 ← sg-app        │
└──────────────────────┘
```

Now the database trusts resources associated with `sg-app`, rather than hard-coding their individual IP addresses. AWS supports SG-to-SG referencing and uses this pattern for tier-to-tier access. ([AWS Documentation][1])

---

# 6. Three-Tier Architecture

This is one of the most important real-world SG designs.

```text
                        INTERNET
                           │
                        443/80
                           │
                           ▼
                 ┌─────────────────┐
                 │      ALB        │
                 │ SG: sg-alb      │
                 └────────┬────────┘
                          │
                       TCP 80
                          │
                          ▼
                 ┌─────────────────┐
                 │   Application   │
                 │      EC2        │
                 │ SG: sg-app      │
                 └────────┬────────┘
                          │
                       TCP 3306
                          │
                          ▼
                 ┌─────────────────┐
                 │    Amazon RDS   │
                 │     MySQL       │
                 │ SG: sg-db       │
                 └─────────────────┘
```

Configure:

```text
sg-alb
Inbound:
443 ← 0.0.0.0/0
80  ← 0.0.0.0/0

sg-app
Inbound:
80 ← sg-alb

sg-db
Inbound:
3306 ← sg-app
```

This creates:

```text
Internet
   │
   ▼
 ALB
   │
   ▼
 EC2
   │
   ▼
 RDS
```

but prevents:

```text
Internet ─────X─────> EC2

Internet ─────X─────> RDS

ALB ──────────X─────> RDS
```

assuming there are no other rules permitting those paths.

That is the significance of **layered SG design**.

---

# 7. Same Security Group on Two EC2 Instances

This is commonly misunderstood.

Suppose:

```text
EC2-A
SG = sg-web

EC2-B
SG = sg-web
```

Architecture:

```text
       sg-web                       sg-web
┌──────────────────┐         ┌──────────────────┐
│      EC2-A       │         │      EC2-B       │
│   10.0.1.10      │         │   10.0.1.20      │
└──────────────────┘         └──────────────────┘
```

Simply attaching the **same SG does not automatically mean arbitrary traffic between the instances is allowed**.

You can explicitly create a self-reference:

```text
Inbound

All Traffic
Source = sg-web
```

Then:

```text
                 sg-web
       ┌─────────────────────────┐
       │                         │
       ▼                         ▼
┌──────────────┐         ┌──────────────┐
│    EC2-A     │◄───────►│    EC2-B     │
│   sg-web     │         │   sg-web     │
└──────────────┘         └──────────────┘
```

AWS specifically documents using the SG ID itself as a source when associated instances need permitted communication. ([AWS Documentation][3])

You don't necessarily need `All Traffic`; better security is often:

```text
TCP 8080 ← sg-app
```

if application nodes only need port 8080 between themselves.

---

# 8. Different Security Groups

Suppose:

```text
EC2-A → sg-A
EC2-B → sg-B
```

Requirement:

```text
EC2-A ---- SSH :22 ----> EC2-B
```

Configure `sg-B`:

```text
Inbound:

SSH
TCP 22
Source = sg-A
```

Architecture:

```text
┌──────────────┐
│    EC2-A     │
│    sg-A      │
└──────┬───────┘
       │
       │ TCP 22
       ▼
┌──────────────┐
│    EC2-B     │
│    sg-B      │
│              │
│ 22 ← sg-A    │
└──────────────┘
```

This is much easier to manage than tracking changing EC2 private IPs.

---

# 9. Bastion Host Architecture

Consider private EC2 servers that should not receive SSH directly from the internet.

```text
                   ADMIN
                     │
                  SSH :22
                     │
                     ▼
              ┌─────────────┐
              │   Bastion   │
              │ sg-bastion  │
              │ Public      │
              └──────┬──────┘
                     │
                  SSH :22
                     │
                     ▼
              ┌─────────────┐
              │ Private EC2 │
              │ sg-private  │
              └─────────────┘
```

Rules:

```text
sg-bastion

Inbound:
22 ← Admin-IP/32


sg-private

Inbound:
22 ← sg-bastion
```

Therefore:

```text
Internet ─────X────> Private EC2

Admin → Bastion → Private EC2
                   ✓
```

For modern management, AWS also recommends considering Systems Manager Session Manager instead of exposing SSH/RDP directly. ([AWS Documentation][4])

---

# 10. ALB → EC2 Architecture

A strong pattern is to prevent users from bypassing the load balancer.

```text
                 INTERNET
                     │
                   HTTPS
                    443
                     │
                     ▼
             ┌──────────────┐
             │     ALB      │
             │    sg-alb    │
             └──────┬───────┘
                    │
                   :80
                    │
             ┌──────┴───────┐
             ▼              ▼
        ┌─────────┐    ┌─────────┐
        │  EC2-1  │    │  EC2-2  │
        │ sg-app  │    │ sg-app  │
        └─────────┘    └─────────┘
```

Rules:

```text
ALB SG
443 ← 0.0.0.0/0

EC2 SG
80 ← sg-alb
```

Instead of:

```text
EC2 SG
80 ← 0.0.0.0/0
```

use:

```text
EC2 SG
80 ← sg-alb
```

This establishes the intended path:

```text
Client → ALB → EC2
```

rather than:

```text
Client ─────────> EC2
```

---

# 11. ALB → EC2 → RDS

A production-style architecture:

```text
                       INTERNET
                          │
                        HTTPS
                          │
                          ▼
                 ┌────────────────┐
                 │      ALB       │
                 │     sg-alb     │
                 └───────┬────────┘
                         │
                      TCP 8080
                         │
               ┌─────────┴─────────┐
               ▼                   ▼
        ┌────────────┐       ┌────────────┐
        │   EC2-1    │       │   EC2-2    │
        │   sg-app   │       │   sg-app   │
        └──────┬─────┘       └──────┬─────┘
               │                    │
               └─────────┬──────────┘
                         │
                      TCP 3306
                         │
                         ▼
                  ┌────────────┐
                  │ Amazon RDS │
                  │   sg-db    │
                  └────────────┘
```

Rules:

```text
sg-alb
443 ← Internet

sg-app
8080 ← sg-alb

sg-db
3306 ← sg-app
```

This is a very important interview and architecture pattern.

---

# 12. RDS Security Group

Avoid this:

```text
MySQL 3306
Source = 0.0.0.0/0
```

Prefer:

```text
MySQL 3306
Source = sg-app
```

Architecture:

```text
Application
EC2 / ECS / etc.
SG = sg-app
       │
       │ 3306
       ▼
┌─────────────────┐
│       RDS       │
│     sg-db       │
│                 │
│ 3306 ← sg-app   │
└─────────────────┘
```

AWS security guidance warns about overly permissive database access and recommends restricting database rules to specific IPs or EC2 security groups. ([AWS Documentation][5])

---

# 13. PostgreSQL Example

```text
Application EC2
      │
      │ TCP 5432
      ▼
PostgreSQL RDS
```

```text
sg-db:

Type: PostgreSQL
Protocol: TCP
Port: 5432
Source: sg-app
```

---

# 14. Jenkins Example

Suppose Jenkins runs on EC2.

```text
Developer
    │
    │ TCP 8080
    ▼
┌───────────────┐
│ Jenkins EC2   │
│ sg-jenkins    │
└───────────────┘
```

Training/lab configuration might be:

```text
8080 ← YOUR-IP/32
22   ← YOUR-IP/32
```

Avoid unnecessarily exposing Jenkins administration to:

```text
8080 ← 0.0.0.0/0
```

---

# 15. SonarQube Example

If SonarQube is running on EC2:

```text
Developer
    │
    │ TCP 9000
    ▼
┌────────────────┐
│ SonarQube EC2  │
│ sg-sonarqube   │
└────────────────┘
```

For a controlled lab:

```text
9000 ← YOUR-IP/32
22   ← YOUR-IP/32
```

If Jenkins needs to contact SonarQube:

```text
Jenkins
sg-jenkins
    │
    │ TCP 9000
    ▼
SonarQube
sg-sonarqube
```

then:

```text
sg-sonarqube

9000 ← sg-jenkins
```

---

# 16. Kubernetes / EKS Concept

The same principle applies to layered application communication:

```text
              INTERNET
                  │
                  ▼
           Load Balancer
                  │
                  ▼
             EKS Nodes
                  │
                  ▼
               Pods
```

Security groups can control applicable network-interface traffic, while Kubernetes networking, Services and NetworkPolicies provide additional controls inside the cluster.

Do not think of an SG as replacing Kubernetes NetworkPolicy.

---

# 17. EFS Example

Typical concept:

```text
EC2
sg-app
   │
   │ NFS TCP 2049
   ▼
EFS Mount Target
sg-efs
```

Configure:

```text
sg-efs

NFS
TCP 2049
Source = sg-app
```

AWS includes EFS among its standard security-group rule use cases. ([AWS Documentation][3])

---

# 18. ICMP / Ping Example

An EC2 instance will not necessarily respond to ping merely because it has a public IP.

You need appropriate ICMP ingress, for example:

```text
Custom ICMP IPv4
Echo Request
Source = YOUR-IP/32
```

Flow:

```text
Laptop
   │
   │ ICMP Echo Request
   ▼
EC2
   │
   │ Echo Reply
   ▼
Laptop
```

AWS documents inbound ICMP Echo Request as the rule required for IPv4 ping. ([AWS Documentation][6])

---

# 19. Security Groups Are Stateful

This is one of the most important concepts.

Suppose:

```text
Inbound:
22 ← YOUR-IP
```

You connect:

```text
Laptop                         EC2
   │                            │
   │──── SSH Request :22 ──────>│
   │                            │
   │<──── Response ─────────────│
```

You do **not** need to create a separate inbound/outbound rule just for every response packet of the established connection.

AWS connection tracking recognizes the response as part of the allowed connection. ([AWS Documentation][2])

Remember:

```text
Security Group
      =
   STATEFUL
```

---

# 20. SG vs Network ACL

This is another important interview question.

| Feature         | Security Group        | Network ACL                  |
| --------------- | --------------------- | ---------------------------- |
| Applied at      | Resource/ENI level    | Subnet level                 |
| Stateful        | **Yes**               | No                           |
| Allow           | Yes                   | Yes                          |
| Explicit Deny   | No                    | Yes                          |
| Rule processing | All applicable rules  | Rule-number order            |
| Return traffic  | Automatically handled | Must be explicitly permitted |
| SG reference    | Yes                   | No SG-to-SG model            |

AWS recommends security groups as the primary resource-level network control and NACLs when subnet-level stateless controls/guardrails are useful. ([AWS Documentation][7])

Architecture:

```text
Internet
   │
   ▼
Internet Gateway
   │
   ▼
┌───────────────────────────┐
│          SUBNET           │
│                           │
│       Network ACL         │
│            │              │
│            ▼              │
│     ┌──────────────┐      │
│     │ Security SG  │      │
│     │      │       │      │
│     │      ▼       │      │
│     │     EC2      │      │
│     └──────────────┘      │
│                           │
└───────────────────────────┘
```

---

# 21. Multiple Security Groups on One EC2

An EC2 network interface can have multiple SGs.

For example:

```text
EC2
 │
 ├── sg-web
 ├── sg-admin
 └── sg-monitoring
```

Conceptually:

```text
sg-web
80,443 allowed

sg-admin
22 allowed from Admin

sg-monitoring
9100 allowed from monitoring servers
```

The allowed rules effectively combine.

So if any associated SG permits matching traffic, another SG doesn't "deny" it because security groups don't contain deny rules.

This is important:

```text
SG-A allows port 22
SG-B has no port 22 rule

Result:

Port 22 can still be allowed through SG-A.
```

---

# 22. Security Groups Do Not Override Routing

This is another major troubleshooting concept.

Suppose:

```text
EC2
SG allows port 80
```

That alone does **not** make the instance internet accessible.

You may also need correct:

```text
VPC
 │
 ├── Subnet
 │
 ├── Route Table
 │
 ├── Internet Gateway
 │
 ├── Public IP / EIP
 │
 ├── Network ACL
 │
 ├── Security Group
 │
 └── Application listening on port
```

Traffic troubleshooting should therefore look like:

```text
Client
   │
   ▼
DNS
   │
   ▼
Route / IGW
   │
   ▼
NACL
   │
   ▼
Security Group
   │
   ▼
EC2
   │
   ▼
OS Firewall
   │
   ▼
Application
```

A rule:

```text
HTTP 80 ← 0.0.0.0/0
```

doesn't help if Apache/nginx isn't actually listening on port 80.

---

# 23. Security Group Referencing Does NOT Copy Rules

Suppose:

```text
sg-A

22 ← 0.0.0.0/0
```

and:

```text
sg-B

8080 ← sg-A
```

This does **not** mean:

```text
sg-B automatically receives:

22 ← 0.0.0.0/0
```

It means resources associated with `sg-A` are accepted as sources for the specified `sg-B` rule.

AWS explicitly states that rules from a referenced SG are **not copied** into the referencing SG. ([AWS Documentation][1])

---

# 24. Network Load Balancer Example

Modern NLBs can have security groups when configured appropriately.

```text
Client
   │
   ▼
┌─────────────┐
│     NLB     │
│   sg-nlb    │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ EC2 Target  │
│  sg-target  │
└─────────────┘
```

AWS recommends that target SGs can reference the NLB SG, allowing traffic through the NLB while preventing direct client access to the targets. ([AWS Documentation][8])

For example:

```text
sg-nlb
443 ← Approved clients

sg-target
443 ← sg-nlb
```

---

# 25. Microservices Security Group Pattern

Consider:

```text
Frontend
   │
   ▼
Orders Service
   │
   ▼
Payment Service
   │
   ▼
Database
```

Create:

```text
sg-frontend
sg-orders
sg-payment
sg-db
```

Rules:

```text
sg-orders
8080 ← sg-frontend

sg-payment
8081 ← sg-orders

sg-db
5432 ← sg-payment
```

Architecture:

```text
┌──────────────┐
│   Frontend   │
│ sg-frontend  │
└──────┬───────┘
       │ 8080
       ▼
┌──────────────┐
│    Orders    │
│  sg-orders   │
└──────┬───────┘
       │ 8081
       ▼
┌──────────────┐
│   Payment    │
│ sg-payment   │
└──────┬───────┘
       │ 5432
       ▼
┌──────────────┐
│ PostgreSQL   │
│    sg-db     │
└──────────────┘
```

This is much stronger than:

```text
Everything → 0.0.0.0/0
```

because the rules express **application relationships**.

---

# 26. Common Wrong vs Correct Configurations

### SSH

Wrong:

```text
22 ← 0.0.0.0/0
```

Better:

```text
22 ← Admin-IP/32
```

or use a controlled management mechanism such as Session Manager.

### Database

Wrong:

```text
3306 ← 0.0.0.0/0
```

Better:

```text
3306 ← sg-app
```

### Application behind ALB

Wrong:

```text
EC2:
8080 ← 0.0.0.0/0
```

Better:

```text
EC2:
8080 ← sg-alb
```

### Internal communication

Less maintainable:

```text
8080 ← 10.0.1.25/32
```

Often better:

```text
8080 ← sg-frontend
```

---

# 27. Practical Port Examples

| Application   | Typical Port | Recommended Source Example |
| ------------- | -----------: | -------------------------- |
| SSH           |           22 | Admin IP / Bastion         |
| HTTP          |           80 | Internet or ALB SG         |
| HTTPS         |          443 | Internet or LB SG          |
| RDP           |         3389 | Admin IP                   |
| MySQL         |         3306 | Application SG             |
| PostgreSQL    |         5432 | Application SG             |
| Jenkins       |         8080 | Admin/network/LB SG        |
| SonarQube     |         9000 | Jenkins/Admin SG           |
| Prometheus    |         9090 | Monitoring/admin SG        |
| Node Exporter |         9100 | Prometheus SG              |
| Grafana       |         3000 | Admin/LB SG                |
| EFS/NFS       |         2049 | Application/EC2 SG         |

These are typical application ports; the correct SG source depends on the architecture.

---

# 28. Complete DevOps Architecture Example

```text
                         INTERNET
                            │
                         HTTPS 443
                            │
                            ▼
                    ┌──────────────┐
                    │     ALB      │
                    │    sg-alb    │
                    └──────┬───────┘
                           │
                        TCP 8080
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
       ┌──────────────┐          ┌──────────────┐
       │    App-1     │          │    App-2     │
       │    sg-app    │          │    sg-app    │
       └──────┬───────┘          └──────┬───────┘
              │                         │
              └────────────┬────────────┘
                           │
                        TCP 3306
                           │
                           ▼
                    ┌──────────────┐
                    │     RDS      │
                    │    sg-db     │
                    └──────────────┘


        ADMIN / DEVOPS
              │
              │ SSH 22
              ▼
        ┌──────────────┐
        │   Bastion    │
        │  sg-bastion  │
        └──────┬───────┘
               │
               │ SSH 22
               ▼
          App Servers
```

Security logic:

```text
sg-alb
443  ← Internet

sg-app
8080 ← sg-alb
22   ← sg-bastion

sg-db
3306 ← sg-app

sg-bastion
22   ← Admin-IP/32
```

The trust chain becomes:

```text
Internet
   │
   ▼
sg-alb
   │
   ▼
sg-app
   │
   ▼
sg-db
```

while administration follows:

```text
Admin
  │
  ▼
sg-bastion
  │
  ▼
sg-app
```

---

# 29. The Most Important Security Group Logic to Remember

```text
                AWS SECURITY GROUP
                       │
       ┌───────────────┴───────────────┐
       │                               │
    INBOUND                         OUTBOUND
       │                               │
       ▼                               ▼
Who can reach me?              Where can I initiate
                               connections to?
```

And remember these six rules:

```text
1. Security Groups are STATEFUL.

2. Security Groups contain ALLOW rules,
   not explicit DENY rules.

3. Inbound rule:
   Source → Resource

4. Outbound rule:
   Resource → Destination

5. SG references are ideal for
   service-to-service/tier-to-tier trust.

6. Same SG attached to two resources
   does not by itself mean all traffic
   between them is permitted.
```

AWS's own VPC guidance treats security groups as the primary resource-level network access mechanism, with NACLs available as a secondary subnet-level control. ([AWS Documentation][7])

For training, the most useful progression is **Single EC2 → Same-SG EC2 → Different SGs → Bastion → ALB/EC2 → ALB/EC2/RDS → NACL vs SG → troubleshooting**. That sequence makes the SG logic much easier for students to understand.

[AWS Security Group Rules documentation](https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html?utm_source=chatgpt.com)

[AWS Security Group Use-Case Examples](https://docs.aws.amazon.com/us_en/AWSEC2/latest/UserGuide/security-group-rules-reference.html?utm_source=chatgpt.com)

[1]: https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html?utm_source=chatgpt.com "Security group rules - Amazon Virtual Private Cloud"
[2]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/security-group-connection-tracking.html?utm_source=chatgpt.com "Amazon EC2 security group connection tracking - Amazon Elastic Compute Cloud"
[3]: https://docs.aws.amazon.com/us_en/AWSEC2/latest/UserGuide/security-group-rules-reference.html?utm_source=chatgpt.com "Security group rules for different use cases - Amazon Elastic Compute Cloud"
[4]: https://docs.aws.amazon.com/prescriptive-guidance/latest/security-controls-by-caf-capability/infrastructure-controls.html?utm_source=chatgpt.com "Security control recommendations for protecting infrastructure - AWS Prescriptive Guidance"
[5]: https://docs.aws.amazon.com/awssupport/latest/user/security-checks.html?utm_source=chatgpt.com "Security - AWS Support"
[6]: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/security-group-rules-reference.html?utm_source=chatgpt.com "Security group rules for different use cases - Amazon Elastic Compute Cloud"
[7]: https://docs.aws.amazon.com/vpc/latest/userguide/infrastructure-security.html?utm_source=chatgpt.com "Infrastructure security in Amazon VPC - Amazon Virtual Private Cloud"
[8]: https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-security-groups.html?utm_source=chatgpt.com "Update the security groups for your Network Load Balancer - Elastic Load Balancing"
