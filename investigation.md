# Full DFIR Investigation Report

## 1. Initial Symptom
<img width="1919" height="659" alt="01_virtualbox_internal_network_configuration" src="https://github.com/user-attachments/assets/c262e424-d565-4082-a19b-84c50f01929b" />

DC01 was unreachable from all Linux-based VMs while Windows-to-Windows communication worked normally.

This suggested a layered network or hypervisor-level issue rather than OS misconfiguration.

---

## 2. Early Hypotheses Tested
<img width="957" height="1030" alt="02_dc01_network_adapter_and_static_ip_configuration" src="https://github.com/user-attachments/assets/0644dbe8-a4b5-47da-976b-922639bf77a0" />

- Firewall issues
- Routing misconfiguration
- IP conflicts
- Wazuh agent misconfiguration
- DNS resolution issues

All were eliminated through testing and did not explain cross-platform failure behavior.

---

## 3. Layer Isolation Process
<img width="723" height="486" alt="03_wazuh_agent_connectivity_asymmetry" src="https://github.com/user-attachments/assets/e5e1e8c2-f1d7-4828-a291-642ffce27f36" />

A structured traffic matrix was created to isolate failure boundaries.

This confirmed:
- Linux ↔ Windows communication failing
- Windows ↔ Windows working
- Linux ↔ Linux working

Conclusion: issue was below OS networking layer.

---

## 4. Key Technical Observations
<img width="961" height="1012" alt="05_wazuh_port_1514_missing" src="https://github.com/user-attachments/assets/57e841d5-71b6-4944-8028-ef789ec88016" />
<img width="967" height="1025" alt="06_wazuh_ports_restored_1514_1515" src="https://github.com/user-attachments/assets/9f27909d-5fd5-4c0d-9769-89481d399a80" />

- Zero packet capture on DC01 NIC in Wireshark
- ARP requests from Wazuh never reached DC01
- ICMP packets never observed at target interface
- OS-level networking stack confirmed idle during failures

---

## 5. VirtualBox-Level Investigation
<img width="957" height="1018" alt="07_conflicting_netplan_ip_configuration" src="https://github.com/user-attachments/assets/c7aa78e2-4e35-4fdd-a823-4577e19d7bab" />

Using VBoxManage debugvm statistics:

- Asymmetric packet delivery observed
- DC01 received inbound traffic correctly
- Outbound DC01 traffic significantly dropped at switch level

This indicated internal switch-level forwarding inconsistency.

---

## 6. Root Cause Analysis
<img width="965" height="1007" alt="08_arp_neighbor_failed_state" src="https://github.com/user-attachments/assets/3b426985-3ddb-4926-9905-3b50383fc2ce" />
<img width="719" height="519" alt="09_arping_zero_response_layer2_failure" src="https://github.com/user-attachments/assets/64a788a9-77df-48e7-8a10-c014abe7760f" />

Evidence strongly indicates a Layer 2 forwarding behavior consistent with stale MAC learning within the VirtualBox internal switch.

This was likely triggered by MAC address changes performed while the switch instance was active.

---

## 7. Resolution
<img width="798" height="314" alt="10_missing_internal_network_interface" src="https://github.com/user-attachments/assets/6ecad7a0-36c0-4d30-b9cc-d204420479e5" />

- DC01 cloned into a fresh VM (clean NIC state)
- Internal network recreated (ADLAB → ADLAB2)

Result:
- Full bidirectional connectivity restored
- 0% packet loss confirmed

---
## 8. Final resolution proof
<img width="704" height="477" alt="12_bidirectional_connectivity_restored" src="https://github.com/user-attachments/assets/724547d3-8144-47a6-8175-dd175cadfa6a" />

## 9. Key Learnings
<img width="1919" height="1030" alt="11_npcap_ndis_binding_disabled" src="https://github.com/user-attachments/assets/e1271fca-cca6-4511-85fe-77858adaa90a" />

- Always validate traffic flow before assuming OS-level failure
- Hypervisor layer must be included in DFIR thinking
- MAC changes during runtime can corrupt virtual switch state
- Packet-level validation is more reliable than OS-level indicators
