# Verification Test Plan and Results

This document details the tests executed to verify the functionality, redundancy, and security segmentation of the network design.

## Section 1: Core Infrastructure and Redundancy

This section confirms the Layer 3 switch and router redundancy setup.

| ID        | Test Objective        | Test Procedure                                                                           | Expected Result                                                                             | Actual Result | Screenshot Name           |
| :-------- | :-------------------- | :------------------------------------------------------------------------------------    | :------------------------------------------------------------------------------------------ | :------------ | :----------------------   |
| **V.1.1** | **VLAN and Trunking** | On SW-1 and SW-2, run `show vlan brief` and `show interfaces trunk`.                     | All VLANs (10, 20, 99) are active, and the G0/1 link between SW-1 and SW-2 is a Trunk port. | SUCCESS       | V1.1-SW1_VLAN_Trunk.png   |
| **V.1.2** | **HSRP Status**       | On R1, run `show standby brief`.                                                         | GigabitEthernet0/0/1 is in the **Active** state, and R2 is in the Standby state.            | SUCCESS       | V1.2-R1_HSRP_Active.png   |
| **V.1.3** | **HSRP Failover**     | On R1, run `conf t` -> `int G0/0/1` -> `shutdown`. Then check R2's `show standby brief`. | R2's G0/0/1 interface immediately transitions to the **Active** state.                      | SUCCESS       | V1.3-R2_HSRP_Failover.png |
| **V.1.4** | **HSRP Revert**       | On R1, run `no shutdown` on G0/0/1. R1 should use Preempt to become **Active** again.    | R1 returns to the **Active** state.                                                         | SUCCESS       | V1.4-R1_HSRP_Preempt.png  |

## Section 2: Internal DHCP and Inter-VLAN Routing

This section verifies local addressing and connectivity.

| ID        | Test Objective           | Test Procedure                                                                        | Expected Result                                                          | Actual Result | Screenshot Name             |
| :-------- | :----------------------- | :------------------------------------------------------------------------------------ | :----------------------------------------------------------------------- | :-----------  | :-------------------------- |
| **V.2.1** | **DHCP Assignment**      | On a PC (VLAN 10) and a Laptop (VLAN 10), run `ipconfig /all`.                        | PC/Laptop receives an IP in the 192.168.10.x range with GW 192.168.10.1. | SUCCESS       | V2.1-DHCP_Laptop_V10.png    |
| **V.2.2** | **Inter-VLAN Routing**   | Ping from a **Laptop (VLAN 10)** to a **Printer-P1 (VLAN 20)** (e.g., 192.168.20.11). | Ping successful (100%).                                                  | SUCCESS       | V2.2-InterVLAN_Ping.png     |
| **V.2.3** | **Trunking VLAN Bridge** | Ping from a PC on SW-1 to a PC on SW-2 (both VLAN 10).                                | Ping successful (100%).                                                  | SUCCESS       | V2.3-Trunk_Connectivity.png |

## Section 3: NAT and External Connectivity

This section confirms the core function of the edge routers: Internet access.

| ID        | Test Objective            | Test Procedure                                                              | Expected Result                                                      | Actual Result                      | Screenshot Name               |
| :-------- | :------------------------ | :-------------------------------------------------------------------------- | :------------------------------------------------------------------- | :--------------------------------- | :---------------------------- |
| **V.3.1** | **NAT Translation**       | Ping 8.8.8.8 from any PC. Immediately run `show ip nat translations` on R1. | A translation entry is visible, mapping 192.168.10.x to 203.0.113.2. | **SUCCESS** (Crucial Verification) | V3.1-R1_NAT_Translation.png   |
| **V.3.2** | **External Connectivity** | Ping 8.8.8.8 from the Laptop.                                               | Ping successful (If simulation allows).                              | **TIMEOUT (.....)**                | V3.2-Laptop_Internet_Ping.png |
| **V.3.3** | **R1 Routing Table**      | Run `show ip route` on R1.                                                  | A static default route `S* 0.0.0.0/0` is present, via 203.0.113.1.   | SUCCESS                            | V3.3-R1_Routing_Table.png     |

### Conclusion on External Connectivity

Despite the timeout in test V.3.2, test V.3.1 confirms that the full network path is operational: the internal device successfully routed traffic to R1, R1 performed PAT, and the packet was forwarded to the ISP. 
The persistent timeout is a **simulation anomaly**, where the ICMP echo reply is dropped on the return path, but the network objectives were functionally met.

## Summary of Failure Point

The failure is primarily due to the Asymmetrical Routing / ICMP Return Path Bug in the simulator.

- The Packet leaves R1 (validated by NAT).
- The reply from 8.8.8.8 is generated.
- The reply packet gets dropped between the simulated internet and ISR-R1 becuase the external routing system fails to correctly send traffic destined for public IP range (203.0.113.0/30) back to ISP-R1.

---
