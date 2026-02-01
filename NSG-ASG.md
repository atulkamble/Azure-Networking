## 🔐 Network Security Group (NSG) vs Application Security Group (ASG) in Azure

![Image](https://learn.microsoft.com/en-us/azure/virtual-network/media/network-security-group-how-it-works/network-security-group-interaction.png)

![Image](https://learn.microsoft.com/en-us/azure/virtual-network/media/security-groups/application-security-groups.png)

![Image](https://docs.microsoft.com/en-us/learn/modules/secure-and-isolate-with-nsg-and-service-endpoints/media/3-exercise-first-task.svg)

### 🔹 Network Security Group (NSG)

**NSG controls network traffic** at the subnet or NIC level using IP-based rules.

**Key points:**

* Works at **Layer 4 (TCP/UDP)**
* Uses **IP addresses, CIDR, ports, protocols**
* Can be attached to:

  * Subnets
  * Network Interfaces (NICs)
* Contains **Inbound & Outbound rules**
* Stateless rules (Azure handles state automatically)

**Example:**

> Allow TCP 443 from Internet → Web Subnet
> Deny all other inbound traffic

---

### 🔹 Application Security Group (ASG)

**ASG is a logical grouping of VMs** to simplify NSG rules.

**Key points:**

* Used **inside NSG rules**
* Groups VMs by **application role** (Web, App, DB)
* Eliminates hard-coding IP addresses
* Improves **readability & manageability**
* Works only within the **same VNet**

**Example:**

> Allow traffic from **Web-ASG → App-ASG on port 8080**

---

## 🆚 Side-by-Side Comparison

| Feature               | NSG                            | ASG                    |
| --------------------- | ------------------------------ | ---------------------- |
| Purpose               | Enforce network security rules | Group VMs logically    |
| Uses IP addresses     | ✅ Yes                          | ❌ No                   |
| Used independently    | ✅ Yes                          | ❌ No (used within NSG) |
| Scope                 | Subnet / NIC                   | VM level               |
| Best for              | Traffic control                | Simplifying NSG rules  |
| Reduces IP management | ❌ No                           | ✅ Yes                  |

---

## 🧠 How They Work Together (Real-World Pattern)

👉 **NSG = Security Rules Engine**
👉 **ASG = Logical VM Grouping**

**Typical 3-Tier App:**

* Web-ASG
* App-ASG
* DB-ASG

**Rules in NSG:**

* Internet → Web-ASG : 80/443
* Web-ASG → App-ASG : 8080
* App-ASG → DB-ASG : 1433
* Deny all else 🚫

---
Perfect 👍 — here’s the **updated version with clear steps + ready-to-use Azure CLI & Terraform code** (AZ-104 / real-world aligned).

---

## 🔐 NSG vs ASG in Azure — **Steps + Codes**

![Image](https://learn.microsoft.com/en-us/azure/bastion/media/bastion-nsg/figure-1.png)

![Image](https://miro.medium.com/1%2Ac93LwmAyXhJobl3S1Mnepw.png)

![Image](https://learn.microsoft.com/en-us/azure/virtual-network/media/manage-network-security-group/view-security-rules.png)

---

## 🧭 Scenario (3-Tier Application)

* **Web Tier** → HTTP/HTTPS
* **App Tier** → Internal API
* **DB Tier** → Database access

We will:

1. Create **VNet + Subnet**
2. Create **ASGs** (Web, App, DB)
3. Create **NSG**
4. Add **NSG rules using ASGs**
5. Associate NSG to Subnet / NIC

---

## ✅ Step 1: Create Resource Group

```bash
az group create \
  --name rg-nsg-asg-demo \
  --location eastus
```

---

## ✅ Step 2: Create Virtual Network & Subnet

```bash
az network vnet create \
  --resource-group rg-nsg-asg-demo \
  --name vnet-demo \
  --address-prefix 10.0.0.0/16 \
  --subnet-name subnet-app \
  --subnet-prefix 10.0.1.0/24
```

---

## ✅ Step 3: Create Application Security Groups (ASG)

```bash
az network asg create \
  --resource-group rg-nsg-asg-demo \
  --name asg-web \
  --location eastus

az network asg create \
  --resource-group rg-nsg-asg-demo \
  --name asg-app \
  --location eastus

az network asg create \
  --resource-group rg-nsg-asg-demo \
  --name asg-db \
  --location eastus
```

📌 **ASGs group VMs logically (no IP dependency)**

---

## ✅ Step 4: Create Network Security Group (NSG)

```bash
az network nsg create \
  --resource-group rg-nsg-asg-demo \
  --name nsg-app \
  --location eastus
```

---

## ✅ Step 5: Create NSG Rules Using ASGs

### 🔹 Allow Internet → Web Tier (HTTP/HTTPS)

```bash
az network nsg rule create \
  --resource-group rg-nsg-asg-demo \
  --nsg-name nsg-app \
  --name Allow-Web-HTTP \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes Internet \
  --destination-asgs asg-web \
  --destination-port-ranges 80 443
```

---

### 🔹 Allow Web → App Tier

```bash
az network nsg rule create \
  --resource-group rg-nsg-asg-demo \
  --nsg-name nsg-app \
  --name Allow-Web-To-App \
  --priority 200 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-asgs asg-web \
  --destination-asgs asg-app \
  --destination-port-ranges 8080
```

---

### 🔹 Allow App → DB Tier

```bash
az network nsg rule create \
  --resource-group rg-nsg-asg-demo \
  --nsg-name nsg-app \
  --name Allow-App-To-DB \
  --priority 300 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-asgs asg-app \
  --destination-asgs asg-db \
  --destination-port-ranges 1433
```

---

## ✅ Step 6: Associate NSG to Subnet

```bash
az network vnet subnet update \
  --resource-group rg-nsg-asg-demo \
  --vnet-name vnet-demo \
  --name subnet-app \
  --network-security-group nsg-app
```

---

## ✅ Step 7: Attach ASG to VM NIC (Important ❗)

```bash
az network nic update \
  --resource-group rg-nsg-asg-demo \
  --name webvm-nic \
  --application-security-groups asg-web
```

👉 Repeat for App VM (`asg-app`) and DB VM (`asg-db`)

---

## 🧠 Terraform Example (Short & Clean)

```hcl
resource "azurerm_application_security_group" "web" {
  name                = "asg-web"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_network_security_rule" "web_http" {
  name                        = "Allow-Web-HTTP"
  priority                    = 100
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_address_prefix       = "Internet"
  destination_application_security_group_ids = [
    azurerm_application_security_group.web.id
  ]
  destination_port_ranges     = ["80", "443"]
  resource_group_name         = azurerm_resource_group.rg.name
  network_security_group_name = azurerm_network_security_group.nsg.name
}
```

---

## 🎯 AZ-104 / Interview One-Liners

* **NSG** → controls *traffic rules*
* **ASG** → controls *VM grouping*
* **ASG cannot exist without NSG**
* **ASG works only inside same VNet**
* **Best practice:** Subnet-level NSG + ASG-based rules

---
