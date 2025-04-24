# Real World Architecture – FiveM Infrastructure

## 1. Public-Facing Security Layer

### Cloudflare (DNS + WAF)
- **Role:**  
  Acts as the first line of defense. Obfuscates the IP-address (outer border) and blocks regular attacks, bottraffic and scans.
- **Features:**
  - DNS resolution
  - HTTPS/TLS
  - Basic rate limiting
  - Web Application Firewall (WAF)

### Loadbalancer (Ubuntu Box 1)
- **Role:**  
  Entry point behind Cloudflare. Handles HTTPS traffic from Cloudflare and load-balances across two reverse proxy servers.
- **Service:**  
  - Runs **nginx** on port `443`
  - HTTPS/TLS
  - Forwards traffic based on `least_conn` or IP hashing to internal proxies
- **Security:**  
  - Firewall configured to accept traffic only from Cloudflare IP ranges
  - No direct public IP exposure for the proxies or origin server

## 2. Reverse Proxy Layer

### Full Reverse Proxy #1 (Ubuntu Box 2)
- **Role:**  
  Handle full reverse proxying to the FiveM game server on TCP/UDP port `30120`.
- **Features:**
  - Transparent forwarding of FiveM protocol (including handshake)
  - Preserves client IP using `proxy_protocol`
  - Configured with firewall rules to only allow traffic from the loadbalancer
### Full Reverse Proxy #2 (Ubuntu Box 3)
    - Same as above, and more reverse proxies can be added for scaling

## 3. Application Layer (Origin)

### FiveM Game Server (Windows Server)
- **Role:**  
  The actual FiveM server instance, hosting the game world.
- **Hosting:**  
  Hosted internally with no public IP access. Protected behind proxies.
- **Access Logic:**
  - Origin only allows traffic from reverse proxy IPs
  - Integrated with custom access control logic
    - Traffic is only allowed through if the connecting player is whitelisted
    - Dynamic firewall rules open UDP/TCP for that client temporarily

## 4. Advanced Defensive Logic

### Access Control Integration
- **Mechanism:**  
  Player authentication (whitelist check) is used as a gatekeeper. Only authenticated and approved player IPs are granted port access.
- **Tools:**  
  - Custom middleware in application layer
  - Integration with FiveM authentication flow
  - Dynamic firewall rules managed via PowerShell on Windows Server

### Rate Limiting & Lockdown Mode
- **Trigger:**  
  Monitors SYN flood patterns.
- **Response:**  
  - When threshold exceeded, lockdown mode activates:
    - Whitelist established connections
    - Block new incoming connections for short interval
- **Implementation:**  
  - PowerShell script running continusly
  - Uses Windows Firewall rules
  - Alert and awareness is provided with Discord webhooks

---

## 5. Monitoring & Visibility

### Prometheus + Node Exporter
- Deployed on proxy and origin servers to monitor system load and connection patterns.

### Grafana
- Dashboards used to track:
  - Incoming SYN/ACK packets
  - Established connections
  - Lockdown triggers
  - IP-based rate metrics

---

## 6. Summary

This architecture was iteratively developed in response to persistent DDoS attacks. By combining application-level access control, network-layer filtering, and dynamic rate limiting, the infrastructure ensures service availability while remaining flexible and adaptive under attack conditions.
