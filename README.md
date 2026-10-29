AWS VPC Peering: Multi-VPC Architecture & Private Connectivity
AWS VPC Peering Terraform Validated Security Audited Project Status

Production-grade implementation of bidirectional, non-transitive AWS VPC Peering between two isolated Virtual Private Clouds. Features automated Infrastructure as Code (Terraform & CloudFormation), least-privilege security group firewalls, an OpenSSH Bastion ProxyJump topology, positive/negative network verification, and clean teardown validation.

📌 Project Overview & Objectives
Problem Statement: In enterprise cloud environments, applications in separate VPCs frequently require secure, low-latency communication without routing traffic across the public internet or exposing backend instances to external attack vectors.
Core Objectives:
Design and provision two non-overlapping VPCs: VPC A (10.10.0.0/16) and VPC B (10.20.0.0/16).
Implement an OpenSSH Bastion Jump Host in VPC A's public subnet to mediate administrative access into private subnets without exposing private workloads.
Establish an active VPC Peering Connection (ewb-a-b-peer) with symmetric Layer 3 route table propagation in both VPCs.
Enforce strict Security Group ingress rules scoped strictly to /32 host IPs for ICMP ping probes and SSH administration.
Empirically validate connectivity through:
Baseline Failure Test (isolation before peering routes)
Forward Path Test (App-A → App-B 0% packet loss)
Reverse Path Test (App-B → App-A 0% packet loss)
Controlled Fault Injection (stateful firewall validation via rule drop)
Recovery Test (instant reachability restoration)
Cleanup Verification (complete resource de-provisioning)
🏛️ Network Architecture & Topology
flowchart TB
    subgraph Internet ["Public Internet / Administrator Workstation"]
        AdminLaptop["Admin Laptop<br/>(Trusted Public IP /32)"]
    end

    subgraph VPCA ["VPC A (10.10.0.0/16) - ap-south-1"]
        direction TB
        IGWA["Internet Gateway (ewb-igw-a)"]

        subgraph SubnetAPub ["Public Subnet (10.10.1.0/24)"]
            Bastion["ewb-bastion<br/>Public IP: Dynamic<br/>Private IP: 10.10.1.10<br/>SG: ewb-sg-bastion (Port 22 from Admin /32)"]
        end

        subgraph SubnetAPriv ["Private Subnet (10.10.2.0/24)"]
            AppA["ewb-app-a<br/>Private IP: 10.10.2.10 (No Public IP)<br/>SG: ewb-sg-app-a (SSH from Bastion /32, ICMP from App-B /32)"]
        end
    end

    subgraph PeeringConn ["VPC Peering Connection (pcx-078073f607a5a75f0)"]
        PeerLink["AWS Private Network Backbone<br/>Non-Transitive / Direct Routing"]
    end

    subgraph VPCB ["VPC B (10.20.0.0/16) - ap-south-1"]
        direction TB
        subgraph SubnetBPriv ["Private Subnet (10.20.1.0/24)"]
            AppB["ewb-app-b<br/>Private IP: 10.20.1.10 (No Public IP, No IGW)<br/>SG: ewb-sg-app-b (SSH from Bastion /32, ICMP from App-A /32)"]
        end
    end

    AdminLaptop -->|SSH Port 22| IGWA
    IGWA --> Bastion
    Bastion -.->|SSH ProxyJump 10.10.1.10| AppA
    Bastion -.->|SSH ProxyJump across PCX| AppB
    AppA <===>|ICMP Ping Probes over PCX| PeerLink
    PeerLink <===>|ICMP Ping Probes over PCX| AppB
P Addressing & Route Table Specifications
Route Table	Associated Subnet	Destination CIDR	Target Gateway / Link	Function
ewb-rt-a-public	ewb-a-public (10.10.1.0/24)	10.10.0.0/16
0.0.0.0/0
10.20.0.0/16	local
ewb-igw-a
pcx-ID	Local VPC A traffic
Default route for Bastion internet access
Enables Bastion to ProxyJump SSH to App-B
ewb-rt-a-private	ewb-a-private (10.10.2.0/24)	10.10.0.0/16
10.20.0.0/16	local
pcx-ID	Local VPC A traffic
Forward path from App-A to VPC B workload
ewb-rt-b-private	ewb-b-private (10.20.1.0/24)	10.20.0.0/16
10.10.0.0/16	local
pcx-ID	Local VPC B traffic
Return path from App-B back to VPC A
