# Attack Chain Diagram

The following diagram illustrates the full attack path from the external attacker (Kali) through the DMZ, into the isolated internal network, and finally to root access on the Analytics server.

```text
+-----------------------------------------------------------------------------------+
|                                 EXTERNAL NETWORK                                  |
|                                                                                   |
|   [ Kali Linux (Attacker) ]                                                       |
|   IP: 10.20.40.10                                                                 |
+----------------------------------------+------------------------------------------+
                                         |
                                         | (1) Exploit Command Injection (DVWA)
                                         |     Payload: 127.0.0.1; bash -c ...
                                         v
+-----------------------------------------------------------------------------------+
|                                    DMZ NETWORK                                    |
|                                                                                   |
|   [ Partner Portal (Docker Container) ]                                           |
|   IP: 10.20.40.20 (External) / 172.17.0.x (Internal)                              |
|   User: www-data                                                                  |
|                                                                                   |
|   * Reverse shell established back to Kali.                                       |
|   * Discovered exposed SSH Key: /var/www/.ssh/id_ed25519                          |
|   * Transferred Ligolo-ng agent to container.                                     |
+----------------------------------------+------------------------------------------+
                                         |
                                         | (2) Establish Ligolo Tunnel (Bypass Segmentation)
                                         |     Proxy on Kali <---> Agent in Container
                                         v
+-----------------------------------------------------------------------------------+
|                              INTERNAL NETWORK (10.20.30.0/24)                     |
|                                                                                   |
|   [ Analytics Server ]                                                            |
|   IP: 10.20.30.40                                                                 |
|                                                                                   |
|   * (3) SSH Login as 'rivermark' using stolen key via tunnel.                     |
|   * (4) Enumerate sudo privileges -> `NOPASSWD: /usr/bin/vim`                     |
|   * (5) Exploit sudo vim -> Spawn root shell (`uid=0`)                            |
|   * (6) Read confidential drafts in /srv/rivermark/drafts/                        |
|                                                                                   |
+-----------------------------------------------------------------------------------+