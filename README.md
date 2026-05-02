# Active Directory DFIR Network Incident Lab

A multi-VM Active Directory security lab built to practice DFIR investigation, detection engineering, and adversary work.

During initial setup, a persistent Layer 2 connectivity failure isolated the domain controller from all Linux VMs.

This repository documents the full investigation — every action taken, every layer eliminated, and the evidence that identified the root cause.

---

## Environment

| VM            | Role               | OS                  | IP              |
|---------------|--------------------|---------------------|-----------------|
| DC01          | Domain Controller  | Windows Server 2022 | 192.168.100.10  |
| WAZUH-SERVER  | SIEM               | Ubuntu Server       | 192.168.100.50  |
| WIN10-CLIENT  | Victim workstation | Windows 10 Pro      | 192.168.100.20  |
| KALI          | Attacker node      | Kali Linux          | 192.168.100.99  |

Virtualization: Oracle VM VirtualBox  
Network: VirtualBox Internal Network (ADLAB)  
Subnet: 192.168.100.0/24

---

## Problem Statement

DC01 was persistently unreachable from all Linux VMs in both directions.  
WIN10 communicated with all machines.

The failure was consistent and symmetric across all testing.

**Traffic matrix — built as the first diagnostic step:**

| Source | Destination | Result |
|--------|-------------|--------|
| Wazuh  | WIN10       | ✅ Pass |
| Wazuh  | DC01        | ❌ Fail |
| Kali   | Wazuh       | ✅ Pass |
| Kali   | DC01        | ❌ Fail |
| DC01   | WIN10       | ✅ Pass |
| DC01   | Wazuh       | ❌ Fail |
| WIN10  | DC01        | ✅ Pass |

Linux↔Linux: works. Windows↔Windows: works. Cross-platform: fails in both directions.

A pattern this clean points to a specific layer rather than general misconfiguration.

---

## Investigation — Layer Elimination

Every failed fix is documented as evidence, not a dead end.

| Fix Attempted | Result | What the Failure Proved |
|---|---|---|
| Firewall disabled (all profiles) | No change | Not OS-level packet filtering |
| Routing table verified and corrected | No change | Not a routing issue |
| Netplan duplicate IP removed | No change | Not an IP conflict on Wazuh |
| Static ARP entries added both sides | No change | Switch drops frames regardless of ARP state |
| Npcap NDIS binding disabled on DC01 | No change | Not primary cause (kept disabled in final state) |
| MAC address changed on DC01 | No change | Corruption persists across MAC changes while switch runs |
| NIC detach and reattach via VBoxManage | No change | Running-state fixes cannot repair corrupted switch state |
| Internal network renamed ADLAB→ADLAB2 | WIN10 worked, DC01 still failed | Confirmed DC01-specific fault, not switch-wide |

---

## Key Finding — Packet Capture Evidence

ARP and ICMP filters active. `arping` running from Wazuh simultaneously.  
Zero packets captured on DC01.

Wazuh's frames were not arriving at DC01's NIC.

The failure was below the OS. The Windows networking stack never saw the traffic.

Every software-level fix from this point was provably irrelevant before it was tried.

---

## Runtime Evidence

After OS-level tools showed nothing, investigation moved to VirtualBox runtime counters.

| Metric | Value |
|---|---|
| Wazuh — packets sent into switch | 1401 |
| DC01 — packets received | 1378 |
| DC01 — packets sent into switch | 200 |
| Wazuh — packets received | 79 |

The switch was accepting DC01's frames and silently discarding ~60% before delivery.

No errors. No logs. No warnings.

---

## Interpretation

The evidence strongly indicates a Layer 2 forwarding behavior consistent with stale MAC learning within the VirtualBox internal switch.

DC01's MAC address had been changed multiple times while the switch instance was running, creating conditions consistent with outdated or orphaned forwarding entries.

Renaming the network confirmed switch-level involvement.

Cloning DC01 confirmed NIC binding isolation was also required.

Both actions together restored full connectivity (0% packet loss).

---

## Resolution

1. **Clone DC01 → DC01-FRESH**
   - Clean NIC binding
   - No association with corrupted switch state

2. **Rename Internal Network → ADLAB2**
   - Forces new VirtualBox switch instance
   - Clears stale forwarding state

Result: full bidirectional connectivity restored.

---

## Key Lessons

- Build a traffic matrix first — it accelerates isolation dramatically
- Zero Wireshark captures = OS layer excluded immediately
- VirtualBox internal switch issues are invisible without runtime counters
- MAC changes must never be done while switch is active
- Rename internal network to reset switch state when behavior is inconsistent

---

## Tools Used

| Tool | Purpose |
|---|---|
| VBoxManage debugvm statistics | Packet-level switch analysis |
| VBoxManage showvminfo | NIC verification |
| VBox.log | Switch initialization tracing |
| arping | Layer 2 ARP validation |
| Wireshark | Packet capture validation |
| ip / netsh tools | ARP + network diagnostics |

---

## Repository Structure
