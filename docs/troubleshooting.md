# Troubleshooting Log

The network layer of this lab didn't work on the first try — twice, in fact, for the same underlying reason on two different segments. This log covers the issues that took real diagnostic work to resolve.

## 1. `emX` interface names didn't match VMware's adapter order

**Symptom:** After assigning WAN/LAN/OPT1 in the OPNsense console (in the order the adapters were added in VMware), nothing on the LAN segment could reach the firewall — but `ping` worked *before* interfaces were reassigned via the console, and stopped working after.

**Diagnosis:** Ran `tcpdump -ni emX` on each interface individually while generating traffic from known sources (ping from Kali, ARP requests). This revealed that `em0` was actually sitting on the NAT (WAN) network, not LAN as assumed — VMware's adapter order in the VM settings UI does **not** guarantee the order FreeBSD enumerates them as `em0`, `em1`, `em2`.

**Fix:** Re-ran interface assignment (`Assign Interfaces` in the OPNsense console) with the *verified* mapping instead of the assumed one.

**Takeaway:** Never trust interface order — verify with traffic capture before configuring anything that depends on it.

## 2. IP address conflicts between host-side virtual adapters and the firewall gateway

**Symptom:** After fixing the interface mapping, LAN traffic still didn't reach the target segment. Pings would either time out or, oddly, appear to succeed inconsistently.

**Diagnosis:** Checked ARP tables on the client and noticed the MAC address answering for the gateway IP had the `00:50:56:xx:xx:xx` prefix — VMware's convention for **host-side** virtual adapters, not `00:0c:29:xx:xx:xx`, which is what actual VMs get. This meant the Windows host itself, not the OPNsense VM, was answering ARP requests for the gateway address. Windows automatically creates a "VMware Network Adapter VMnetX" for every host-only network, and it had picked up the same `.1` address that had just been assigned to the OPNsense interface on that segment.

**Fix:** Reassigned the host-side adapter's IP off the conflicting address (e.g., `10.10.10.1` → `10.10.10.254`) on both affected segments, leaving `.1` exclusively for OPNsense.

**Takeaway:** When building host-only networks in VMware, always check (and if necessary reassign) the IP on the automatically created host-side adapter *before* assigning the same subnet's gateway address elsewhere.

## 3. Diagnostic method that ended up working reliably

For any "traffic isn't reaching X" problem in this lab, this sequence consistently isolated the fault:

1. `ping` — confirm basic L3 reachability
2. `arp -n` on both ends — confirm L2 resolves to the *expected* MAC (watch for `00:50:56` vs `00:0c:29` prefixes — that distinction directly exposed both IP conflicts above)
3. `sockstat -4 -l` on OPNsense — confirm the target service is actually listening
4. `pfctl -sr`, then temporarily `pfctl -d` — rule out (or confirm) the firewall as the cause
5. `pfctl -ss` — check whether a state entry is actually being created for the connection
6. `tcpdump -ni <interface>`, run *live* alongside the test (not reviewed after the fact) — confirms whether packets are arriving on a given interface at all
7. `tcpdump -eni <interface>` (with MAC addresses) — the step that actually revealed the IP conflicts, by showing replies addressed to the wrong MAC

This stepwise elimination — L2 before L3, state table before rules, live capture before logs — is the main practical skill this project reinforced.
