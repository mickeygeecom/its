# Lab Architecture

## 1. Physical and Virtual Infrastructure

### OVH Dedicated Server
- **Description:**  
  The physical host with a single public IP address (e.g., `193.70.34.141`). This server is hosted at OVH and provides the hardware resources for the lab.

### Proxmox Hypervisor
- **Description:**  
  Installed on the OVH dedicated server to virtualize and manage multiple VMs.
- **Configuration:**  
  Proxmox is configured with two virtual bridges:
  - **vmbr0 (Public Network):**
    - **IP Assignment:**  
      Holds the server’s public IP (`193.70.34.141/24`).
    - **Alias:**  
      Has a secondary alias IP `192.168.110.1/24` that serves as the internal gateway for the WAN side of OPNsense.
  - **vmbrlab (Internal Lab Network):**
    - **IP Assignment:**  
      A dedicated internal network for the lab VMs, using the subnet `192.168.1.0/24`.

## 2. Virtual Machines (VMs)

### OPNsense Firewall VM
- **Role:**  
  Acts as the central security and routing device for the lab.
- **Interfaces:**
  - **WAN Interface:**
    - **Connection:**  
      Attached to `vmbr0` (Public Network).
    - **IP Assignment:**  
      Configured with a private IP, e.g., `192.168.110.254/24`.
    - **Upstream Gateway:**  
      Set to the alias IP `192.168.110.1` on vmbr0 (which Proxmox NATs to the public IP).
    - **External Access:**  
      Proxmox DNAT rules forward external port (e.g., `8443`) on the public IP to port `443` on OPNsense’s WAN.
  - **LAN Interface:**
    - **Connection:**  
      Attached to `vmbrlab` (Internal Lab Network).
    - **IP Assignment:**  
      Static IP, e.g., `192.168.1.1/24`.
- **Functions:**
  - Performs NAT and routing, ensuring lab VMs can access the internet.
  - Acts as a firewall between the internal lab (attacker, victim, monitoring tools) and outbound traffic.
  - Provides the web management interface (WebGUI) for configuration and monitoring.

### Kali Linux (Attacker) VM
- **Connection:**  
  Deployed on the `vmbrlab` network.
- **IP Assignment:**  
  Acquires an IP via DHCP from OPNsense in the `192.168.1.x` range (e.g., `192.168.1.10`).
- **Role:**  
  Used for penetration testing and offensive operations in the lab.

### Ubuntu Server (Victim) VM
- **Connection:**  
  Also deployed on the `vmbrlab` network.
- **IP Assignment:**  
  Receives an IP via DHCP from OPNsense in the `192.168.1.x` range (e.g., `192.168.1.20`).
- **Role:**  
  Serves as the target system to simulate vulnerabilities in the environment.

### Additional Monitoring/Incident Response Tools
- **Future Deployment:**  
  Additional VMs or containers for tools such as Wazuh, Prometheus, Grafana, and Suricata can be deployed on `vmbrlab` and integrated into the lab.

## 3. Outbound and External Access

### Outbound Traffic
- **Flow:**
  - Lab VMs (Kali, Ubuntu, etc.) send traffic to the internet through OPNsense.
  - OPNsense NATs the internal lab network (`192.168.1.0/24`) out over its WAN interface (`192.168.110.254/24`).
  - Proxmox uses an iptables SNAT rule to translate traffic from the `192.168.110.x` subnet to the public IP (`193.70.34.141`).

### External Management (DNAT)
- **Setup:**
  - A DNAT rule on Proxmox forwards incoming connections on a chosen external port (e.g., `8443`) on the public IP to OPNsense’s WAN port `443`.
- **Result:**
  - This allows remote access to the OPNsense WebGUI from outside your network by browsing to:  
    `https://proxmox.mickeygee.com:8443/`
