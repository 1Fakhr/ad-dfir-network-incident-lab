# Full DFIR Investigation Report

## 1. Initial Symptom
DC01 was unreachable from all Linux-based VMs while Windows-to-Windows communication worked normally.

This suggested a layered network or hypervisor-level issue rather than OS misconfiguration.

---

## 2. Early Hypotheses Tested

- Firewall issues
- Routing misconfiguration
- IP conflicts
- Wazuh agent misconfiguration
- DNS resolution issues

All were eliminated through testing and did not explain cross-platform failure behavior.

---

## 3. Layer Isolation Process

A structured traffic matrix was created to isolate failure boundaries.

This confirmed:
- Linux ↔ Windows communication failing
- Windows ↔ Windows working
- Linux ↔ Linux working

Conclusion: issue was below OS networking layer.

---

## 4. Key Technical Observations

- Zero packet capture on DC01 NIC in Wireshark
- ARP requests from Wazuh never reached DC01
- ICMP packets never observed at target interface
- OS-level networking stack confirmed idle during failures

---

## 5. VirtualBox-Level Investigation

Using VBoxManage debugvm statistics:

- Asymmetric packet delivery observed
- DC01 received inbound traffic correctly
- Outbound DC01 traffic significantly dropped at switch level

This indicated internal switch-level forwarding inconsistency.

---

## 6. Root Cause Analysis

Evidence strongly indicates a Layer 2 forwarding behavior consistent with stale MAC learning within the VirtualBox internal switch.

This was likely triggered by MAC address changes performed while the switch instance was active.

---

## 7. Resolution

- DC01 cloned into a fresh VM (clean NIC state)
- Internal network recreated (ADLAB → ADLAB2)

Result:
- Full bidirectional connectivity restored
- 0% packet loss confirmed

---

## 8. Key Learnings

- Always validate traffic flow before assuming OS-level failure
- Hypervisor layer must be included in DFIR thinking
- MAC changes during runtime can corrupt virtual switch state
- Packet-level validation is more reliable than OS-level indicators
