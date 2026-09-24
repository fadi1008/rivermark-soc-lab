24-9-2026 : 
Decision: Use VirtualBox Internal Network rivermark-lan (10.20.30.0/24) for attack traffic between Kali and the portal.

Reason: Isolation from home LAN, no accidental scanning of real devices, controlled environment.

Temporary NAT: Only for updates. Never Bridged on the attacker VM.