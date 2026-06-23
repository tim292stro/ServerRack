## Section 1. Master Architecture Topology

```mermaid
graph TD
    subgraph Public WAN Layer [Public Internet Edge]
        VIP[Floating Public WAN VIP: 203.0.113.10]
        eth0_1[Server 1 Public WAN: eth0]
        eth0_2[Server 2 Public WAN: eth0]
        VIP <--> eth0_1
        VIP <--> eth0_2
    end

    subgraph Out-of-Band Sync Layer [Direct Interconnect]
        eth200g_1[Server 1 200GbE Interconnect: eth200g]
        eth200g_2[Server 2 200GbE Interconnect: eth200g]
        eth200g_1 <== GlusterFS + Corosync Link 0 ==> eth200g_2
    end

    subgraph Internal Enterprise Core [High-Availability Network Blocks]
        bond0_1[Server 1 10G LACP Bond: bond0]
        bond0_2[Server 2 10G LACP Bond: bond0]
        LAN_VIP[Floating LAN/DMZ VIP: 10.0.0.1 / Class A]
        CORE_SW[Internal LAN Core Switch]
        bond0_1 <--> LAN_VIP
        bond0_2 <--> LAN_VIP
        LAN_VIP <--> CORE_SW
    end

    subgraph Point-to-Point Management Fabric [Class C /30 Subnets]
        KVM_1[Server 1 IPMI/KVM Interface: 192.168.99.2/30]
        KVM_2[Server 2 IPMI/KVM Interface: 192.168.99.6/30]
        MINT_2[Linux Mint Interface 2: 192.168.99.1/30]
        MINT_3[Linux Mint Interface 3: 192.168.99.5/30]
        MINT_2 <== Direct Cable run ==> KVM_1
        MINT_3 <== Direct Cable run ==> KVM_2
    end

    subgraph Central Control Station [Central Orchestration Unit]
        MINT_BOX[Linux Mint Management PC]
        GPS[u-blox ZED-X20P GNSS] -->|NMEA Data Stream| MINT_BOX
        RB[SRS PRS10 Rubidium Standard] -->|Disciplined 1PPS Signal| MINT_BOX
        MINT_BOX <--> MINT_2
        MINT_BOX <--> MINT_3
    end
```

### Infrastructure Network Matrix

| Network Path | Interface Role | Subnet / Mask | Mint Side Host IP | Server Target IP |
| :--- | :--- | :--- | :--- | :--- |
| **KVM Link 1** | Server 1 Boot / Time / QDevice | `192.168.99.0/30` | `192.168.99.1` | `192.168.99.2` |
| **KVM Link 2** | Server 2 Boot / Time / QDevice | `192.168.99.4/30` | `192.168.99.5` | `192.168.99.6` |
| **Interconnect** | Distributed Storage Sync Loop | `10.200.0.0/24` | — | `.1` (S1) / `.2` (S2) |
| **Internal LAN** | VM Bridging & Host Routing | `10.0.0.0/8` | — | `.11` (S1) / `.12` (S2) |
| **DMZ Network** | Isolated Guest Allocations | `172.16.0.0/16` | — | Dynamic VM Routing |
| **Public WAN** | Redundant External Uplinks | *ISP Assigned* | — | Assigned IPs / Floating VIP |
