# Master Deployment Manual: FOSS High-Availability Cluster

This master deployment manual details the end-to-end setup of a 100% Free and Open-Source Software (**FOSS**), subscription-free High-Availability network front-end and hosting environment. 

This architecture leverages a **Linux Mint** management desktop orchestrating two headless, diskless **Debian 12** servers. The headless nodes boot their core operating systems over isolated, point-to-point **Class C /30 subnets** via the Supermicro IP-KVM Virtual Media interface. 

A dedicated, hardware-disciplined Stratum 1 timing stack runs on the Linux Mint PC using a **u-blox ZED-X20P multi-band GNSS receiver** paired with a **Stanford Research Systems (SRS) PRS10 Rubidium Frequency Standard**. **SatPulse** acts as the high-precision driver frontend while **Chrony** serves time across the isolated networks.

---

Jump to sections:

[Section 1. Master Architecture Topology](Section1.md)
[Section 2. Linux Mint Management Machine Configuration](Section2.md)
[Section 3. Diskless Provisioning Sequence](Section3.md)
[Section 4. Headless Server Architecture & Storage Setup](Section4.md)
[Section 5. High-Availability Cluster Core Setup](Section5.md)
[Section 6. Shorewall HA Configuration Files](Section6.md)
[Section 7. High-Performance RAM-Warmed VM Engine](Section7.md)
[Section 8. FOSS Application Stack Detailed Configuration](Section8.md)
[Section 9. Machine-to-Machine Orchestration Strategy](Section9.md)
[Section 10. Operational Status Check](Section10.md)
[Section 11. Advanced Anti-Tamper Security: GPS Geofenced Hardware Security Module](Section11.md)
[Section 12. Advanced Management Workstation Control: Programmatic Template Engine & Core Playbook Execution](Section12.md)
[Section 13. Real-Time Security Telemetry Cluster Alerts](Section13.md)
[Section 14. Final Cluster Verification & Operational Health Checks](Section14.md)
