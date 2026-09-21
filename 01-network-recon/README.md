Target: Windows 11 VM
Target IP: 192.168.163.4
Attacker/test machine: Kali Linux
Kali IP: 192.168.163.3

Network: 192.168.163.0/24

Initial scan:
nmap 192.168.163.4

Result:
5357/tcp open

Service detection:
Microsoft HTTPAPI httpd 2.0
SSDP/UPnP

Observation:
999 other TCP ports were filtered.

Finding: Windows Function Discovery exposes TCP/5357 to the local subnet when the network profile is Private.

Root cause: Windows Firewall rule NETDIS-WSDEVNT-In-TCP-Active permits inbound TCP/5357 from LocalSubnet.

Validation: Disabling the rule changed Nmap's result from open to filtered.

Risk: Limited in this lab because the Windows VM is isolated on a host-only network and the rule is restricted to the local subnet.

Lesson: Firewall rules can be validated experimentally rather than assuming they are responsible based solely on their names.