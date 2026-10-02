# New Page

## Remote Peering Connection (RPC)

Remote Peering Connection (RPC) is an OCI networking component that enables private network connectivity between Dynamic Routing Gateways (DRGs) located in different OCI Regions.

RPC is primarily used to establish inter-region communication between VCNs through their respective DRGs. It allows private traffic to travel between networks in different regions without requiring communication over the public Internet.

### How RPC Works

RPC works by connecting two DRGs located in different OCI Regions.

```text
VCN
 |
DRG
 |
RPC
 ||
OCI Regional / Inter-Region Network
 ||
RPC
 |
DRG
 |
VCN
```

The basic communication flow is:

1. A VCN is attached to a DRG.
2. An RPC is created on each DRG.
3. The two RPCs are connected using the required remote peering information.
4. Once the RPC connection reaches the `PEERED` state, the DRGs can exchange traffic.
5. Appropriate route rules are required to direct traffic toward the remote network through the DRG.
6. Security controls such as Security Lists or NSGs must allow the required traffic.

### Key Characteristics of RPC

- RPC provides private inter-region connectivity.
- RPC connects DRG-to-DRG, rather than directly connecting VCN-to-VCN.
- RPC is used when networks are located in different OCI Regions.
- Communication can use private IP addresses.
- The RPC connection must be in the `PEERED` state for connectivity.
- Proper routing configuration is required for traffic to reach the remote network.
- Security Lists and NSGs can control which traffic is permitted.
- RPC is commonly used for Disaster Recovery, business continuity, multi-region architectures, and inter-region application connectivity.

### RPC vs Local VCN Peering

| Feature | Local Peering | Remote Peering |
| --- | --- | --- |
| Connectivity | Within the same Region | Between different Regions |
| Main Component | DRG / local peering | DRG / RPC |
| Purpose | Regional VCN connectivity | Inter-region connectivity |
| Private Communication | Yes | Yes |
| Common Use Case | Hub-and-Spoke networking | DR, Multi-Region architecture |

## OCI Disaster Recovery & Inter-Hub Connectivity

### 1. Project Objective

Design and implement a Disaster Recovery environment in a separate OCI Region and establish private inter-region connectivity between the existing Primary Hub VCN and the newly created DR Hub VCN using DRG.

### 2. Existing Resources

The following resources are available in the Primary Region and will be considered as the baseline for the DR design.

**Primary Region:** `US-WEST (Phoenix)`

#### 2.1 Primary Hub VCN

| Parameter | Details |
| --- | --- |
| Compartment | `Hub_Network_Compartment` |
| VCN Name | `Hub_vcn` |
| CIDR Block | `10.0.0.0/24` |
| Subnets | public subnet (`10.0.0.0/25`) |
| Route Table | `hub_publicsub_rt` |
| Security List | `hub_publicsub_sl` |
| Gateway | `Hub_igw` (Internet Gateway) |

#### 2.2 Primary Spoke VCN

| Parameter | Details |
| --- | --- |
| Compartment | `amazon_network_compartment` |
| VCN Name | `amazon_vcn` |
| CIDR Block | `10.0.1.0/24` |
| Subnets | `amazon_db_subnet` (`10.0.1.160/27`)<br>`amazon_web_subnet` (`10.0.1.128/27`)<br>`amazon_app_pvt_subnet` (`10.0.1.0/25`) |
| Route Tables | `amazon_rt`<br>`amazon_db_rt`<br>`amazon_web_rt` |
| Security Lists | `amazon_db_sub_sl`<br>`amazon_web_sub_sl`<br>`amazon_sl` |
| Gateways | `web_igw` (Web Application Internet Gateway)<br>`nat_gw` (Nat Gateway) |

#### 2.3 Primary DRG

| Parameter | Details |
| --- | --- |
| Compartment | `Hub_Network_Compartment` |
| DRG Name | `hub_drg` |
| Hub VCN Attachment | `hub_vcn_attachment` → `Hub_vcn` |
| Spoke VCN Attachment | `amazon_vcn_attachment` → `amazon_vcn` |

### 3. Planning Phase

The Disaster Recovery environment will be designed in a separate OCI Region to provide geographic separation from the existing Primary environment. The DR architecture will mirror the required network structure of the Primary environment while using non-overlapping CIDR ranges.

#### 3.1 Region Planning

**Primary Region**

| Parameter | Details |
| --- | --- |
| Region | US West (Phoenix) |
| Purpose | Primary / Production Environment |

**DR Region**

The DR environment will be deployed in a separate OCI Region from the Primary Region.

| Parameter | Details |
| --- | --- |
| Region | US-EAST (Ashburn) |
| Purpose | Disaster Recovery Environment |
| Geographic Separation | Separate from Primary Region |
| Primary Region | US West (Phoenix) |

**Planning Considerations**

The DR Region will be selected based on:

- Geographic separation from the Primary Region
- Availability of required OCI services
- Availability of required Compute shapes
- Network connectivity requirements
- Expected latency
- Cost considerations
- DR and business continuity requirements

#### 3.2 DR Hub VCN Planning

The existing Primary environment contains a dedicated Hub VCN in the `Hub_Network_Compartment`. To maintain a similar architecture, a dedicated DR Hub VCN will be created in the DR environment.

**DR Hub VCN**

| Parameter | Planned Details |
| --- | --- |
| Compartment | `hub_network_compartment` |
| Region | US-EAST (Ashburn) |
| VCN Name | `dr_hub_vcn` |
| CIDR Block | `10.0.2.0/24` |
| Subnet | `dr_hub_public_subnet` (`10.0.2.0/25`) |
| Route Table | `dr_hub_rt` |
| Security List | `dr_hub_publicsub_sl` |
| Gateway | `Hub_igw` (Internet Gateway) |
| Purpose | Central network hub for the DR environment |

**CIDR Planning**

- Existing Primary Hub: `10.0.0.0/24`
- Existing Primary Spoke: `10.0.1.0/24`
- Therefore, the DR Hub will use: `10.0.2.0/24`

This avoids CIDR overlap with the existing Primary environment.

#### 3.3 DR Spoke VCN Planning

The existing Primary Spoke VCN `amazon_vcn` contains separate Web, Application and Database subnets. To maintain a similar application architecture in the DR environment, a corresponding DR Spoke VCN will be created.

**DR Spoke VCN**

| Parameter | Planned Details |
| --- | --- |
| Compartment | `amazon_network_compartment` |
| Region | US-EAST (Ashburn) |
| VCN Name | `dr_amazon_vcn` |
| CIDR Block | `10.0.3.0/24` |
| Subnets | `dr_amazon_db_subnet` (`10.0.3.160/27`) — private<br>`dr_amazon_web_subnet` (`10.0.3.128/27`) — public<br>`dr_amazon_app_pvt_subnet` (`10.0.3.0/25`) — private |
| Route Tables | `dr_amazon_app_rt`<br>`dr_amazon_db_rt`<br>`dr_amazon_web_rt` |
| Security Lists | `dr_amazon_db_sub_sl`<br>`dr_amazon_web_sub_sl`<br>`dr_amazon_app_sl` |
| Gateways | `dr_web_igw` (Web Application Internet Gateway)<br>`dr_nat_gw` (Nat Gateway)<br>`dr_sgw` (service gateway) |
| Purpose | Hosts the DR application workload |

**CIDR Planning**

- Existing Primary Spoke: `10.0.1.0/24`
- Planned DR Hub: `10.0.2.0/24`
- Therefore, the DR Spoke will use: `10.0.3.0/24`

This avoids CIDR overlap with the existing Primary environment.

#### 3.4 DRG & Attachment Planning

The existing Primary environment uses a DRG (`hub_drg`) to provide connectivity between the Primary Hub VCN and Primary Spoke VCN. A separate DRG will be created in the DR Region to provide connectivity between the DR Hub VCN and DR Spoke VCN. The Primary and DR DRGs will later be connected using inter-region DRG peering to establish private connectivity between the two regions.

| Parameter | Planned Details |
| --- | --- |
| Compartment | `Hub_Network_Compartment` |
| DRG Name | `dr_hub_drg` |
| Region | US East (Ashburn) |
| Purpose | Provides centralized routing and connectivity for the DR environment |
| Hub VCN Attachment | `dr_hub_vcn_attachment` → `dr_hub_vcn` |
| Spoke VCN Attachment | `dr_amazon_vcn_attachment` → `dr_amazon_vcn` |

### 4. Implementation Phase

#### 4.1 Subscribe to the DR Region

**Objective:** Enable the Disaster Recovery region so that OCI resources can be created in the DR environment.

**Configuration**

| Parameter | Value |
| --- | --- |
| Primary Region | US West (Phoenix) |
| DR Region | US East (Ashburn) |

**Steps**

1. Log in to the OCI Console.
2. From the top-right corner, open the Region selector and select **Manage Regions**.
3. Search for US East (Ashburn).
4. Click **Subscribe**.
5. Wait until the region subscription is completed.
6. Open the Region selector again.
7. Select US East (Ashburn).
8. Verify that the region is accessible and OCI resources can be created.

**Validation**

- US East (Ashburn) is available in the Region selector.
- You can access Networking services in the DR region.
- The required compartments are available or can be created in the DR region.

#### 4.2 Create DR Hub VCN

**Objective:** Create the DR Hub VCN with the required subnet, route table, security list, and gateway configuration.

##### Step 1 — Create DR Hub VCN

1. Log in to the OCI Console.
2. Switch to `US East (Ashburn)`.
3. Go to **Networking → Virtual Cloud Networks**.
4. Select compartment: `Hub_Network_Compartment`.
5. Click **Create VCN**.
6. Enter:
   - Name: `dr_hub_vcn`
   - IPv4 CIDR Block: `10.0.2.0/24`
7. Create the VCN.

##### Step 2 — Create DR Hub Route Table

1. Open `dr_hub_vcn`.
2. Go to **Resources → Route Tables**.
3. Click **Create Route Table**.
4. Enter the name `dr_hub_rt`.
5. Create the route table.

At this stage, the route table can remain empty because the DRG and inter-region routing will be configured later.

##### Step 3 — Create DR Hub Security List

1. Open `dr_hub_vcn`.
2. Go to **Resources → Security Lists**.
3. Click **Create Security List**.
4. Enter the name `dr_hub_sl`.
5. Configure the required ingress and egress rules.

##### Step 4 — Create DR Hub Public Subnet

1. Go to **Subnets** under `dr_hub_vcn`.
2. Click **Create Subnet**.
3. Enter:

| Parameter | Value |
| --- | --- |
| Subnet Name | `dr_hub_pubsub` |
| CIDR Block | `10.0.2.0/25` |
| Route Table | `dr_hub_rt` |
| Security List | `dr_hub_sl` |
| Subnet Type | Regional |

4. Create the subnet.

##### Step 5 — Configure Internet Gateway

Since this is a public subnet, create an Internet Gateway if the DR Hub requires direct internet access.

1. Open `dr_hub_vcn`.
2. Go to **Gateways** and click **Create Internet Gateway**.
3. Name it `dr_hub_igw`.
4. Ensure the gateway is enabled.
5. Create it.

##### Step 6 — Add Internet Gateway Route

Open **dr_hub_vcn → Route Tables → dr_hub_rt** and add:

| Destination CIDR | Target Type | Target |
| --- | --- | --- |
| `0.0.0.0/0` | Internet Gateway | `dr_hub_igw` |

#### 4.3 Create DR Spoke VCN and Subnets

**Objective:** Create the DR Spoke VCN to host the DR application, web, and database workloads.

##### 4.3.1 Create DR Spoke VCN

1. Go to **Networking → Virtual Cloud Networks**.
2. Select `amazon_network_compartment`.
3. Click **Create VCN**.
4. Enter:
   - VCN Name: `dr_amazon_vcn`
   - CIDR: `10.0.3.0/24`
5. Create the VCN.

##### 4.3.2 Create Route Tables

Create three separate Route Tables for App, Web, and DB subnets.

**App Route Table**

Name: `dr_amazon_app_rt`. For now, keep it empty. DRG routes will be added later.

**Web Route Table**

Name: `dr_amazon_web_rt`. For the Web subnet, if internet access is required, the appropriate Internet/NAT Gateway route will be added later based on the final design.

**DB Route Table**

Name: `dr_amazon_db_rt`. Keep it empty initially. Required private routes will be added later.

##### 4.3.3 Create Security Lists

Create separate Security Lists for each subnet.

**1. App Security List**

Name: `dr_amazon_app_sl`.

Initial purpose:

- Allow required application and internal traffic.
- Keep only the required ingress/egress rules.

**2. Web Security List**

Name: `dr_amazon_web_sl`.

Required ingress can include:

| Protocol | Port | Purpose |
| --- | --- | --- |
| TCP | 80 | HTTP |
| TCP | 443 | HTTPS |
| TCP | 22 | SSH management, if required |

**3. DB Security List**

Name: `dr_amazon_db_sl`.

For Oracle Database:

| Protocol | Port | Purpose |
| --- | --- | --- |
| TCP | 1521 | Oracle Database |

##### 4.3.4 Create Internet Gateway and NAT Gateway

Create both gateways in `dr_amazon_vcn` to provide internet connectivity for public and private workloads.

**A. Create Internet Gateway**

1. Open `dr_amazon_vcn`.
2. Go to **Internet Gateways**.
3. Click **Create Internet Gateway**.
4. Enter the name `dr_web_igw`.
5. Click **Create**.

**Purpose:** Provide internet connectivity for resources in the public Web subnet.

**B. Create NAT Gateway**

1. Open `dr_amazon_vcn`.
2. Go to **NAT Gateways** and click **Create NAT Gateway**.
3. Enter the name `dr_nat_gw`.
4. Click **Create**.

**Purpose:** Provide outbound internet access for private App and DB workloads without exposing them to the internet.

**Gateway Route Configuration**

After creating both gateways:

**Web Route Table — `dr_amazon_web_rt`**

| Target Type | Destination | Target |
| --- | --- | --- |
| Internet Gateway | `0.0.0.0/0` | Internet Gateway → `dr_amazon_igw` |

**App Route Table — `dr_amazon_app_rt`**

| Target Type | Destination | Target |
| --- | --- | --- |
| NAT Gateway | `0.0.0.0/0` | NAT Gateway → `dr_amazon_nat` |

##### 4.3.5 Create Application, Web and Database Subnets

After creating the DR Spoke VCN and required network components, create the Application, Web, and Database subnets.

| Parameter | Application Subnet | Web Subnet | Database Subnet |
| --- | --- | --- | --- |
| Subnet Name | `dr_amazon_app_pvt_subnet` | `dr_amazon_web_subnet` | `dr_amazon_db_subnet` |
| CIDR | `10.0.3.0/25` | `10.0.3.128/27` | `10.0.3.160/27` |
| Subnet Type | Regional | Regional | Regional |
| Route Table | `dr_amazon_app_rt` | `dr_amazon_web_rt` | `dr_amazon_db_rt` |
| Security List | `dr_amazon_app_sl` | `dr_amazon_web_sl` | `dr_amazon_db_sl` |

**Steps**

1. Go to **Networking → Virtual Cloud Networks** and open `dr_amazon_vcn`.
2. Navigate to **Subnets → Create Subnet**.
3. Enter the required Subnet Name and CIDR Block according to the above configuration.
4. Select **Regional** as the subnet type.
5. Select the corresponding Route Table:
   - Application → `dr_amazon_app_rt`
   - Web → `dr_amazon_web_rt`
   - Database → `dr_amazon_db_rt`
6. Select the corresponding Security List:
   - Application → `dr_amazon_app_sl`
   - Web → `dr_amazon_web_sl`
   - Database → `dr_amazon_db_sl`
7. Click **Create Subnet**.
8. Repeat the same process for the remaining Web and Database subnets using their respective configurations.

**Result:** The DR Spoke VCN is divided into separate Application, Web, and Database subnets, providing logical network segmentation for the DR environment.

#### 4.4 Create DRG and Attach DR VCNs

**Objective:** Create a DRG in the DR region and attach the DR Hub VCN and DR Spoke VCN to provide centralized private connectivity within the DR environment and prepare for inter-region connectivity.

##### 4.4.1 Create DRG

**Configuration**

| Parameter | Value |
| --- | --- |
| Region | US East (Ashburn) |
| Compartment | `Hub_network_compartment` |
| DRG Name | `dr_hub_drg` |

**Steps**

1. Ensure the OCI Console is set to US East (Ashburn).
2. Go to **Networking → Dynamic Routing Gateways**.
3. Select compartment `Hub_Network_Compartment`.
4. Click **Create Dynamic Routing Gateway**.
5. Enter the name `dr_hub_drg`.
6. Click **Create Dynamic Routing Gateway**.
7. Wait until the DRG state becomes **Available**.

##### 4.4.2 Attach DR Hub VCN to the DRG

The DR Hub VCN will be attached to `dr_hub_drg`.

**Configuration**

| Parameter | Value |
| --- | --- |
| VCN | `dr_hub_vcn` |
| DRG | `dr_hub_drg` |
| Attachment Name | `dr_hub_vcn_attachment` |

**Steps**

1. Go to **Networking → Virtual Cloud Networks**.
2. Open `dr_hub_vcn`.
3. Go to **Resources → DRG Attachments**.
4. Click **Create DRG Attachment**.
5. Select DRG `dr_hub_drg`.
6. Enter attachment name `dr_hub_vcn_attachment`.
7. Click **Create**.
8. Wait until the attachment state becomes **Attached**.

##### 4.4.3 Attach DR Spoke VCN to the DRG

The DR Spoke VCN will also be attached to the same DRG.

**Configuration**

| Parameter | Value |
| --- | --- |
| VCN | `dr_amazon_vcn` |
| DRG | `dr_hub_drg` |
| Attachment Name | `dr_amazon_drg_attachment` |

**Steps**

1. Go to **Networking → Virtual Cloud Networks**.
2. Open `dr_amazon_vcn`.
3. Go to **Resources → DRG Attachments**.
4. Click **Create DRG Attachment**.
5. Select DRG `dr_hub_drg`.
6. Enter attachment name `dr_amazon_vcn_attachment`.
7. Click **Create**.
8. Wait until the attachment state becomes **Attached**.

#### 4.5 Configure Security Lists and Route Tables for Connectivity Between DR Hub and DR Spoke VCN

Configure the required Security List rules and Route Table rules to establish private communication between the DR Hub VCN and DR Spoke VCN through the DRG, while retaining the existing Internet Gateway, NAT Gateway, and Service Gateway configurations.

##### 4.5.1 Configure DR Hub Security List

Security List: `dr_hub_sl`  
Subnet: `dr_hub_pubsub`  
CIDR: `10.0.2.0/25`

**Ingress Rules**

| Source CIDR | Protocol | Destination Port | Purpose |
| --- | --- | --- | --- |
| `223.185.43.45/32` | TCP | 22 | Allow SSH access from the management IP |
| `0.0.0.0/0` | TCP | 80 | Allow HTTP traffic, if required |

**Egress Rules**

The existing default egress rule is retained:

| Destination | Protocol | Purpose |
| --- | --- | --- |
| `0.0.0.0/0` | All Protocols | Allow outbound traffic |

##### 4.5.2 Configure DR Hub Route Table

Route Table: `dr_hub_rt`  
Subnet: `dr_hub_pubsub`

Retain the existing Internet Gateway route and add the required subnet-specific DRG route.

| Destination CIDR | Target | Purpose |
| --- | --- | --- |
| `0.0.0.0/0` | `hub_igw` | Allow outbound internet traffic |
| `10.0.3.0/25` | `dr_hub_drg` | Route traffic to the DR App subnet |

##### 4.5.3 Configure DR Application Security List

Security List: `dr_amazon_app_sl`  
Subnet: `dr_amazon_app_pvt_subnet`  
CIDR: `10.0.3.0/25`

**Ingress Rules**

| Source CIDR | Protocol | Destination Port | Purpose |
| --- | --- | --- | --- |
| `10.0.2.0/25` | TCP | 22 | Allow SSH from DR Hub |
| `10.0.2.0/25` | ICMP | All | Allow connectivity testing from DR Hub |
| `10.0.3.128/27` | TCP | 80 | Allow HTTP traffic from DR Web |

**Egress Rules**

| Destination | Protocol | Purpose |
| --- | --- | --- |
| `0.0.0.0/0` | All Protocols | Allow outbound traffic |

##### 4.5.4 Configure DR Application Route Table

Route Table: `dr_amazon_app_rt`  
Subnet: `dr_amazon_app_pvt_subnet`

Retain the existing NAT Gateway and Service Gateway routes.

| Destination CIDR | Target | Purpose |
| --- | --- | --- |
| `0.0.0.0/0` | `dr_nat_gw` | Outbound internet access for private App instances |
| `10.0.2.0/25` | `dr_hub_drg` | Route traffic to the DR Hub subnet |
| All IAD Services | `dr_amazon_sgw` | Access to OCI services |

##### 4.5.5 Configure DR Web Security List

Security List: `dr_amazon_web_sl`  
Subnet: `dr_amazon_web_subnet`  
CIDR: `10.0.3.128/27`

**Ingress Rules**

| Source CIDR | Protocol | Destination Port | Purpose |
| --- | --- | --- | --- |
| `0.0.0.0/0` | TCP | 80 | Allow HTTP traffic |
| `0.0.0.0/0` | TCP | 443 | Allow HTTPS traffic, if required |

**Egress Rules**

| Destination | Protocol | Purpose |
| --- | --- | --- |
| `0.0.0.0/0` | All Protocols | Allow outbound traffic |

##### 4.5.6 Configure DR Web Route Table

Route Table: `dr_amazon_web_rt`  
Subnet: `dr_amazon_web_subnet`

Keep the existing Internet Gateway route. If the Web subnet needs connectivity to the DR Hub subnet, add:

| Destination CIDR | Target | Purpose |
| --- | --- | --- |
| `0.0.0.0/0` | Internet Gateway | Internet access |
| `10.0.2.0/25` | `dr_hub_drg` | Route traffic to the DR Hub subnet |

##### 4.5.7 Configure DR Database Security List

Security List: `dr_amazon_db_sl`  
Subnet: `dr_amazon_db_subnet`  
CIDR: `10.0.3.160/27`

**Ingress Rule**

| Source CIDR | Protocol | Destination Port | Purpose |
| --- | --- | --- | --- |
| `10.0.3.0/25` | TCP | 1521 | Allow Oracle DB traffic from the DR App subnet |

**Egress:** No additional custom rule is required if the default egress rule is retained.

##### 4.5.8 Configure DR Database Route Table

Route Table: `dr_amazon_db_rt`  
Subnet: `dr_amazon_db_subnet`

Since the DB subnet does not currently require communication with the DR Hub and the Primary DB Route Table also has no custom routes, no additional custom DRG route is configured for the DR Database subnet at this stage.

#### 4.6 Test Hub-to-Spoke Connectivity

After configuring the DR Hub and DR Spoke Security Lists, Route Tables, and DRG attachments, connectivity between the DR Hub and DR Spoke was tested to verify that the configured network path was working as expected.

##### 4.6.1 Connectivity Test from DR Hub Server to DR Spoke Server

The connectivity test was performed from the DR Hub Bastion/Server to the DR Spoke Application Server using the private IP address.

**Test Command**

```bash
ping <DR_Spoke_Server_Private_IP>
```

**Expected Result:** The DR Spoke server should respond successfully to ICMP requests.

**Observed Result:** The DR Hub server successfully received responses from the DR Spoke server, confirming that private network connectivity between the Hub and Spoke VCNs was working through the DRG.

##### 4.6.2 SSH Connectivity Test

After successful Ping testing, SSH connectivity was also tested to verify TCP port 22 communication.

**Test Command**

```bash
ssh opc@<DR_Spoke_Server_Private_IP>
```

**Expected Result:** The SSH connection should be established successfully and the user should be able to log in to the DR Spoke server.

This test validates:

- DRG routing between Hub and Spoke
- Security List rule for TCP/22
- Network-level reachability
- SSH service availability on the Spoke server

#### 4.7 Create Remote Peering Connections

**Objective:** Create Remote Peering Connections (RPCs) under the respective DRGs in the Primary and DR regions. These RPCs will later be used to establish private inter-region connectivity between the Primary DRG and DR DRG.

##### 4.7.1 Create RPC on the Primary DRG

Region: `US West (Phoenix)`

1. Navigate to **Networking → Dynamic Routing Gateways (DRG)**.
2. Open the existing DRG `hub_drg`.
3. Navigate to **DRG Attachments**.
4. Under **RPC Attachments**, click **Create Remote Peering Connection**.
5. Enter the required details:

| Parameter | Value |
| --- | --- |
| Name | `primary_to_dr_rpc` |
| Compartment | `Hub_network_compartment` |
| DRG | `hub_drg` |

6. Click **Create**.

The RPC attachment will be created under the `hub_drg` DRG.

**Expected Result:** The RPC should be created successfully and its lifecycle state should show **Available**.

**Note:** At this stage, only the RPCs have been created. The RPCs are not yet connected to each other.

##### 4.7.2 Create RPC on the DR DRG

Switch to `US East (Ashburn)`.

1. Navigate to **Networking → Dynamic Routing Gateways (DRG)**.
2. Open the DR DRG `dr_hub_drg`.
3. Navigate to **DRG Attachments**.
4. Under **RPC Attachments**, click **Create Remote Peering Connection**.
5. Enter:

| Parameter | Value |
| --- | --- |
| Name | `dr_to_primary_rpc` |
| Compartment | `Hub_network_compartment` |
| DRG | `dr_hub_drg` |

6. Click **Create**.

The RPC attachment will be created under the `dr_hub_drg` DRG.

**Expected Result:** The RPC should be created successfully and its lifecycle state should show **Available**.

**Note:** At this stage, only the RPCs have been created. The RPCs are not yet connected to each other.

##### 4.7.3 Establish RPC Connection Between Primary and DR Regions

After creating the RPCs on both the Primary and DR DRGs, establish the remote peering connection between them.

**Step 1: Copy the DR RPC OCID**

Switch to US East (Ashburn) and navigate to **Networking → Dynamic Routing Gateways → dr_hub_drg → DRG Attachments → RPC Attachments**.

1. Open the RPC `dr_to_primary_rpc`.
2. Copy its OCID.
3. Keep the OCID available for the connection establishment step.

The Remote Peering Connection OCID of the DR-side RPC is required when establishing the connection from the Primary side.

**Step 2: Establish the Connection from Primary Region**

Switch back to `US West (Phoenix)`.

1. Navigate to **Networking → Dynamic Routing Gateways → hub_drg**.
2. Go to **DRG Attachments → RPC Attachments**.
3. Select `primary_to_dr_rpc`.
4. Click **Establish Connection**.

The following details are required:

| Parameter | Value |
| --- | --- |
| Region | US East (Ashburn) |
| Remote peering connection OCID | OCID of `dr_to_primary_rpc` |

5. Enter/select the Region as US East (Ashburn).
6. Paste the Remote Peering Connection OCID copied from the DR RPC.
7. Review the details.
8. Click **Establish Connection**.

**Step 3: Verify RPC Peering Status**

After the connection is established, verify the status of the RPC on both sides.

Expected Peering Status: **PEERED**

| Region | DRG | RPC | Status |
| --- | --- | --- | --- |
| US West (Phoenix) | `hub_drg` | `primary_to_dr_rpc` | PEERED |
| US East (Ashburn) | `dr_hub_drg` | `dr_to_primary_rpc` | PEERED |

#### 4.8 Configure Route Tables and Security Lists for Inter-Region Connectivity

After establishing the RPC connection between the Primary and DR DRGs, configure the required VCN Route Table and Security List rules to allow private communication between the Primary Hub VCN and DR Hub VCN.

##### 4.8.1 Configure Primary Hub VCN

Region: `US West (Phoenix)`  
Route Table: `hub_publicsub_rt`

1. Navigate to **Networking → Virtual Cloud Networks**.
2. Open `Hub_vcn`.
3. Go to **Route Tables → hub_publicsub_rt**.
4. Select **Route Rules → Add Route Rules**.
5. Add:

| Parameter | Value |
| --- | --- |
| Destination CIDR | `10.0.2.0/24` |
| Target Type | Dynamic Routing Gateway |
| Target | `hub_drg` |
| Description | Route traffic from the Primary Hub VCN to the DR Hub VCN through the DRG. |

6. Click **Add Route Rules**.

Security List: `hub_publicsub_sl`

7. Go to **Security Lists → hub_publicsub_sl**.
8. Under **Ingress Rules**, click **Add Ingress Rules**.
9. Add:

| Source CIDR | Protocol | Destination Port | Purpose |
| --- | --- | --- | --- |
| `10.0.2.0/25` | ICMP | All | Allow ICMP traffic from the DR Hub for connectivity testing. |
| `10.0.2.0/25` | TCP | 22 | Allow SSH traffic from the DR Hub to the Primary Hub server. |

10. Click **Add Ingress Rules**.

##### 4.8.2 Configure DR Hub VCN

Region: US East (Ashburn)  
Route Table: `dr_hub_rt`

1. Navigate to **Networking → Virtual Cloud Networks**.
2. Open `dr_hub_vcn`.
3. Go to **Route Tables → dr_hub_rt**.
4. Select **Route Rules → Add Route Rules**.
5. Add:

| Parameter | Value |
| --- | --- |
| Destination CIDR | `10.0.0.0/24` |
| Target Type | Dynamic Routing Gateway |
| Target | `dr_hub_drg` |
| Description | Route traffic from the DR Hub VCN to the Primary Hub VCN through the DRG. |

6. Click **Add Route Rules**.

Security List: `dr_hub_publicsub_sl`

7. Go to **Security Lists → dr_hub_publicsub_sl**.
8. Under **Ingress Rules**, click **Add Ingress Rules**.
9. Add:

| Source CIDR | Protocol | Destination Port | Purpose |
| --- | --- | --- | --- |
| `10.0.0.0/25` | ICMP | All | Allow ICMP traffic from the Primary Hub for connectivity testing. |
| `10.0.0.0/25` | TCP | 22 | Allow SSH traffic from the Primary Hub to the DR Hub server. |

10. Click **Add Ingress Rules**.

**Result**

The required Route Table and Security List rules are configured in both regions to support private communication between:

`Primary Hub VCN 10.0.0.0/24 ↔ DR Hub VCN 10.0.2.0/24`

through the DRG and `PEERED` RPC connection.

#### 4.9 Test Primary Hub to DR Hub Connectivity

After completing the RPC peering, Route Table configuration, and Security List configuration, connectivity between the Primary Hub server and DR Hub server was tested using ICMP and SSH.

##### 4.9.1 Ping Connectivity Test

Log in to the Primary Hub Bastion server and run:

```bash
ping <DR_Hub_Bastion_Private_IP>
```

**Expected Result:** The DR Hub Bastion should respond successfully to ICMP requests.

**Result:** The Primary Hub Bastion successfully received responses from the DR Hub Bastion, confirming network-level connectivity between the Primary and DR Hub VCNs.

##### 4.9.2 SSH Connectivity Test

From the Primary Hub Bastion server, test SSH connectivity to the DR Hub Bastion:

```bash
ssh -i <keypath> opc@<DR_Hub_Bastion_Private_IP>
```

**Expected Result:** The SSH connection should be established successfully and the user should be able to log in to the DR Hub Bastion server.

This validates:

- Inter-region routing through the DRG and RPC
- TCP port 22 connectivity
- Security List configuration
- Private IP communication between the Primary and DR Hub servers
