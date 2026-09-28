# Troubleshooting Notes — Milestone 2 Implementation

## 1. Cloud-PT WAN emulation could not bridge the off-site link

**What was attempted:** The original design (per Milestone 1 diagrams) used a Cloud-PT
device to represent the Internet/WAN between the router (R1) and the off-site
administrator's PC, with both ports set to a DSL provider network.

**What went wrong:** Packet Tracer's Cloud-PT DSL configuration page only supports
pairing a Modem-type port with an Ethernet-type port (simulating a real DSL modem's
two sides). It has no mechanism to bridge two plain Ethernet ports directly to each
other, which is what a simple point-to-point WAN link required here. As a result, no
traffic could pass between R1 and the off-site PC — pings and SSH attempts both timed
out with no response.

**Fix applied:** The Cloud-PT device was replaced with a standard, unconfigured 2960
switch (labelled `WAN`). An unconfigured switch bridges all connected ports by
default, requiring no special pairing configuration. R1's Gi0/2 and the off-site PC
were each connected to a separate port on this switch, restoring full connectivity.

**Why this is still a valid design choice:** The Cloud device's only purpose was to
abstract "the wider network between the branch and the off-site administrator" — it
was never meant to represent a literal physical link. A plain switch performs the
same abstraction role without the DSL modem-pairing limitation, and does not affect
the VLAN segmentation (the assigned technical challenge) or the Finance isolation
constraint, both of which are entirely internal to the branch's own switches and
router subinterfaces.

## 2. Off-site PC's wireless connection required a specific NIC module

**What was attempted:** To better represent a realistic remote-access scenario
(CR9), the off-site administrator's connection to the network was upgraded from a
wired link to a wireless one, using an Access Point (AP-PT) between the WAN switch
and the off-site laptop.

**What went wrong:** Packet Tracer's default Laptop-PT ships with a wired
FastEthernet module. Attempting to open the wireless configuration screen returned
an error stating a WMP300N or WPC300N wireless interface was required.

**Fix applied:** The laptop was powered off, its wired module removed, and a
WPC300N wireless module (the laptop-specific wireless card, as opposed to WMP300N
which is for desktop PCs) installed in its place. After powering back on, a wireless
profile was created matching the Access Point's SSID (`Bokamoso-Remote`) and
WPA2-PSK passphrase, and the static IP address was re-applied to the new wireless
interface.

**Result:** The off-site PC successfully associated with the Access Point (full
signal strength and link quality), and both `ping` and `ssh` tests to R1 succeeded
over the wireless link, with the round-trip times visibly higher than the original
wired test — a realistic reflection of wireless latency.

## 3. Finance isolation ACL direction

**What was attempted:** An extended ACL (`BLOCK_FINANCE`) was applied to R1's
Finance subinterface (Gi0/0.20) with `ip access-group BLOCK_FINANCE in`, intended to
block general-user VLANs from reaching the Finance subnet.

**What went wrong:** Testing with `ping 10.15.20.10` from PC-Mgmt succeeded, proving
the ACL was not actually filtering the traffic. The `in` direction filters traffic
*entering* the router from the Finance subinterface (i.e. traffic *leaving* Finance),
not traffic *destined toward* Finance.

**Fix applied:** The ACL was reapplied in the `out` direction instead
(`ip access-group BLOCK_FINANCE out`), which filters traffic as it exits the router
toward the Finance subnet. Retesting confirmed all four general-user VLANs (Mgmt,
Tellers, Loans, Guest) are correctly blocked from reaching Finance, while Finance
devices can still communicate with each other, and control pings to other gateways
remain unaffected.

## Verification summary

| Test | Result |
|---|---|
| PC-Mgmt → Finance PC | Blocked (100% loss, "Destination host unreachable") |
| PC-Mgmt → own gateway | Success (control test) |
| New Finance PC (added post-expansion) → blocked from Mgmt | Blocked (confirms ACL scales by subnet, not per-host) |
| Finance PC → Finance PC | Success (same-VLAN traffic unaffected) |
| Off-site PC → R1 WAN interface (wireless) | Success |
| Off-site PC → SSH into R1 (CR9) | Success, over wireless link |
