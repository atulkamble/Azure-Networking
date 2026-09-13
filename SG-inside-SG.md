## Security Group Inside Security Group — Inbound Rule Logic

When you put **another Security Group as the Source** of an inbound rule, AWS is not putting one SG physically “inside” another. It means:

> **Allow traffic from network interfaces/resources associated with the referenced Security Group, on the specified protocol and port.**

### Example

Two EC2 instances:

```text
EC2-A
SG = sg-web

EC2-B
SG = sg-app
```

Configure `sg-app`:

```text
Inbound Rules — sg-app

Type        Port       Source
--------------------------------
Custom TCP  8080       sg-web
```

Architecture:

```text
              sg-web                       sg-app
        ┌────────────────┐           ┌────────────────┐
        │     EC2-A      │           │     EC2-B      │
        │                │           │                │
        │  10.0.1.10     │── :8080 ─▶│  10.0.2.10     │
        └────────────────┘           └────────────────┘
                                            ▲
                                            │
                                     Inbound Rule
                                     8080 ← sg-web
```

The logic is:

```text
Is traffic arriving at EC2-B?
        │
        ▼
Does EC2-B have sg-app?
        │
       YES
        ▼
Does sg-app allow destination port 8080?
        │
       YES
        ▼
Is the source associated with sg-web?
        │
       YES
        ▼
      ALLOW
```

So:

```text
EC2-A (sg-web) ─── TCP 8080 ───▶ EC2-B (sg-app)
                                  ✅ ALLOWED
```

But:

```text
EC2-C (sg-other) ── TCP 8080 ──▶ EC2-B (sg-app)
                                  ❌ NOT ALLOWED
```

assuming no other rule permits EC2-C.

### Very Important: SG Reference Does NOT Inherit Rules

Suppose:

```text
sg-web
Inbound:
80  ← 0.0.0.0/0
443 ← 0.0.0.0/0
```

and:

```text
sg-app
Inbound:
8080 ← sg-web
```

This **does not mean**:

```text
Internet
   │
   │ 8080
   ▼
sg-web
   │
   ▼
sg-app
```

And it does **not mean** `sg-app` inherits the rules of `sg-web`.

Instead:

```text
Internet ───────────────X──────────────▶ EC2-B :8080

EC2-A associated
with sg-web ───────── :8080 ──────────▶ EC2-B
                                           ✅
```

The referenced SG identifies **eligible source resources**, not traffic that previously passed through that SG.

### Same Security Group Referencing Itself

You can also configure:

```text
sg-app

Inbound:
TCP 8080
Source: sg-app
```

Then resources associated with `sg-app` can communicate on port `8080`:

```text
                    sg-app
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
     EC2-A           EC2-B          EC2-C
        │              │              │
        └────── :8080 ─┼──────────────┘
                       │
                    ALLOWED
```

This is called **self-referencing a Security Group**.

### Real-World Three-Tier Example

```text
                    INTERNET
                       │
                    HTTPS 443
                       ▼
                 ┌───────────┐
                 │    ALB    │
                 │  sg-alb   │
                 └─────┬─────┘
                       │
                     :8080
                       ▼
                 ┌───────────┐
                 │    EC2    │
                 │  sg-app   │
                 └─────┬─────┘
                       │
                     :3306
                       ▼
                 ┌───────────┐
                 │    RDS    │
                 │   sg-db   │
                 └───────────┘
```

Rules:

```text
sg-alb
443  ← 0.0.0.0/0

sg-app
8080 ← sg-alb

sg-db
3306 ← sg-app
```

Read these rules in plain English:

```text
443 ← Internet
"Internet can connect to ALB on 443"

8080 ← sg-alb
"Resources using sg-alb can connect
 to application resources on 8080"

3306 ← sg-app
"Resources using sg-app can connect
 to database resources on 3306"
```

### Key Concept

```text
Inbound Rule:

PORT ← SOURCE SECURITY GROUP
```

means:

```text
"Allow this PORT from resources
 associated with this SOURCE SG."
```

**It does not mean:**

```text
SG-A is inside SG-B                 ❌
SG-B inherits SG-A rules            ❌
Internet allowed to SG-A can
automatically access SG-B           ❌
All traffic from SG-A is allowed    ❌
```

It means only:

```text
Resource associated with SG-A
             │
             │ specified port
             ▼
Resource associated with SG-B
             ✅
```

This **SG-to-SG referencing** is one of the most important AWS patterns for **ALB → EC2 → RDS** architectures.

# AWS Security Group-to-Security Group Communication — Practice Lab

This lab demonstrates **both important cases**:

1. **Self-referencing SG** — resources using the same SG communicate with each other.
2. **Different SG reference** — a destination SG allows traffic specifically from resources associated with another SG.

## Lab Architecture

```text
                    AWS VPC
                 10.0.0.0/16
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼

     EC2-A           EC2-B           EC2-C
   10.0.1.10       10.0.1.20       10.0.1.30

   sg-common        sg-common         sg-app
        │               │               ▲
        │               │               │
        └──── :8080 ────┘               │
          SELF REFERENCE                 │
                                        │
           EC2-A ───── :9090 ───────────┘
                 sg-common → sg-app
```

Use private IP addresses for all SG-to-SG tests.

## Practice 1 — Self-Referencing Security Group

Create:

```text
Security Group: sg-common
```

Attach `sg-common` to:

```text
EC2-A
EC2-B
```

Initially, **do not add** a self-reference inbound rule.

On EC2-B, start a test web server:

```bash
python3 -m http.server 8080
```

Or:

```bash
sudo dnf install -y python3
python3 -m http.server 8080
```

From EC2-A:

```bash
curl http://<EC2-B-PRIVATE-IP>:8080
```

Expected:

```text
Connection timeout / failure
❌
```

Having the **same SG attached to both instances does not automatically permit communication**.

Now add this rule to `sg-common`:

```text
Inbound Rules — sg-common

Type        Protocol    Port     Source
------------------------------------------------
Custom TCP  TCP         8080     sg-common
```

Architecture:

```text
                    sg-common
                       │
              Self Reference :8080
                       │
             ┌─────────┴─────────┐
             │                   │
             ▼                   ▼

           EC2-A ─── TCP 8080 ─▶ EC2-B
             │                   │
         sg-common           sg-common

                         ✅ ALLOWED
```

Test again from EC2-A:

```bash
curl http://<EC2-B-PRIVATE-IP>:8080
```

Expected:

```text
Directory listing...
```

or the page being served by Python.

### Reverse Test

Because both instances use `sg-common`, start the server on EC2-A:

```bash
python3 -m http.server 8080
```

From EC2-B:

```bash
curl http://<EC2-A-PRIVATE-IP>:8080
```

Expected:

```text
✅ ALLOWED
```

The self-reference means:

```text
TCP 8080 ← sg-common
```

Therefore:

```text
EC2-A (sg-common) ──8080──▶ EC2-B (sg-common)  ✅

EC2-B (sg-common) ──8080──▶ EC2-A (sg-common)  ✅
```

---

# Practice 2 — One Security Group Referenced by Another

Now create another SG:

```text
sg-app
```

Attach it to:

```text
EC2-C
```

Configuration:

```text
EC2-A → sg-common
EC2-B → sg-common
EC2-C → sg-app
```

Start a server on EC2-C:

```bash
python3 -m http.server 9090
```

From EC2-A:

```bash
curl http://<EC2-C-PRIVATE-IP>:9090
```

Expected initially:

```text
❌ FAILED
```

Now configure `sg-app`:

```text
Inbound Rules — sg-app

Type        Protocol    Port     Source
------------------------------------------------
Custom TCP  TCP         9090     sg-common
```

This means:

```text
9090 ← sg-common
```

or:

> Resources associated with `sg-common` may connect to resources protected by `sg-app` on TCP 9090.

Architecture:

```text
       Source Resources                    Destination

          sg-common                         sg-app
    ┌───────────────────┐              ┌───────────────┐
    │                   │              │               │
    │ EC2-A             │              │    EC2-C      │
    │ 10.0.1.10         │──── :9090 ──▶│ 10.0.1.30     │
    │                   │              │               │
    │ EC2-B             │              │               │
    │ 10.0.1.20         │──── :9090 ──▶│               │
    │                   │              │               │
    └───────────────────┘              └───────────────┘
                                              ▲
                                              │
                                       9090 ← sg-common
```

Run from EC2-A:

```bash
curl http://<EC2-C-PRIVATE-IP>:9090
```

Expected:

```text
✅ SUCCESS
```

Run from EC2-B:

```bash
curl http://<EC2-C-PRIVATE-IP>:9090
```

Expected:

```text
✅ SUCCESS
```

Both work because both source instances are associated with `sg-common`.

---

# Practice 3 — Prove That SG Reference Is Port-Specific

Keep this rule:

```text
sg-app

9090 ← sg-common
```

Start another server on EC2-C:

```bash
python3 -m http.server 8080
```

From EC2-A:

```bash
curl http://<EC2-C-PRIVATE-IP>:8080
```

Expected:

```text
❌ FAILED
```

But:

```bash
curl http://<EC2-C-PRIVATE-IP>:9090
```

Expected:

```text
✅ SUCCESS
```

This proves:

```text
sg-common → sg-app
```

does **not** mean:

```text
Allow ALL traffic from sg-common ❌
```

It means:

```text
Allow TCP 9090 from sg-common ✅
```

---

# Practice 4 — Add an Unauthorized Instance

Create:

```text
EC2-D
SG = sg-other
```

Architecture:

```text
EC2-A (sg-common) ─── :9090 ───▶ EC2-C (sg-app)
                                  ✅ SUCCESS


EC2-B (sg-common) ─── :9090 ───▶ EC2-C (sg-app)
                                  ✅ SUCCESS


EC2-D (sg-other) ──── :9090 ───▶ EC2-C (sg-app)
                                  ❌ FAILED
```

Test from EC2-D:

```bash
curl http://<EC2-C-PRIVATE-IP>:9090
```

Expected:

```text
❌ timeout
```

This is a strong demonstration that the rule is based on the **source resource's SG association**, not simply its subnet.

---

# Final Practice Architecture

```text
                         VPC 10.0.0.0/16

              ┌──────────────────────────────┐
              │                              │
              │          sg-common           │
              │                              │
              │    ┌────────┐  ┌────────┐   │
              │    │ EC2-A  │  │ EC2-B  │   │
              │    └───┬────┘  └───┬────┘   │
              │        │           │         │
              │        └─ :8080 ───┘         │
              │          SELF SG             │
              │                              │
              └──────────────┬───────────────┘
                             │
                             │ TCP 9090
                             │
                             ▼
                    ┌──────────────────┐
                    │      sg-app      │
                    │                  │
                    │      EC2-C       │
                    │                  │
                    └──────────────────┘

                     Inbound sg-app
                     9090 ← sg-common
```

The two concepts you should demonstrate during the lab are:

```text
SELF REFERENCE

sg-common
8080 ← sg-common

EC2-A ───8080───▶ EC2-B
   sg-common          sg-common
                     ✅


SG → OTHER SG

sg-app
9090 ← sg-common

EC2-A ───9090───▶ EC2-C
   sg-common          sg-app
                     ✅
```

### Recommended test matrix

| Source | Source SG | Destination | Destination SG | Port | Result |
| ------ | --------- | ----------- | -------------- | ---: | ------ |
| EC2-A  | sg-common | EC2-B       | sg-common      | 8080 | ✅      |
| EC2-B  | sg-common | EC2-A       | sg-common      | 8080 | ✅      |
| EC2-A  | sg-common | EC2-C       | sg-app         | 9090 | ✅      |
| EC2-B  | sg-common | EC2-C       | sg-app         | 9090 | ✅      |
| EC2-A  | sg-common | EC2-C       | sg-app         | 8080 | ❌      |
| EC2-D  | sg-other  | EC2-C       | sg-app         | 9090 | ❌      |

This gives you a clean hands-on demonstration of **Same SG → Same SG**, **SG-A → SG-B**, **allowed-port vs blocked-port**, and **authorized SG vs unauthorized SG** behavior.

