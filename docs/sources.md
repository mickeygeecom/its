
# Kilder og Dokumentation

Herunder er en liste over relevante værktøjer, dokumentation og guides anvendt i projektet.

## Virtualisering og Infrastruktur

- **Proxmox VE**  
  Virtualiseringsplatform brugt til laboratorieopsætning.  
  [https://pve.proxmox.com/pve-docs/](https://pve.proxmox.com/pve-docs/)

- **OPNsense Firewall**  
  Bruges som firewall og gateway i laboratoriet.  
  [https://docs.opnsense.org/](https://docs.opnsense.org/)

- **Ubuntu Server**  
  Operativsystem brugt til proxy og loadbalancer-maskiner.  
  [https://ubuntu.com/server/docs](https://ubuntu.com/server/docs)

- **Windows Server 2025**  
  Host for FiveM origin server.  
  [https://learn.microsoft.com/en-us/windows-server/](https://learn.microsoft.com/en-us/windows-server/)

## Pentest og testmiljø

- **Kali Linux**  
  Bruges til simulering af angreb i lab.  
  [https://www.kali.org/docs/](https://www.kali.org/docs/)

- **FiveM**  
  Multiplayer platform, udsat for real-world DDoS-angreb.  
  [https://fivem.net/](https://fivem.net/)

## Overvågning og incident response

- **Prometheus**  
  Metrics collection.  
  [https://prometheus.io/docs/](https://prometheus.io/docs/)

- **Grafana**  
  Dashboard og visualisering.  
  [https://grafana.com/docs/grafana/latest/](https://grafana.com/docs/grafana/latest/)

- **Node Exporter**  
  Metrics agent til Prometheus.  
  [https://github.com/prometheus/node_exporter](https://github.com/prometheus/node_exporter)

- **Medium Guide** – Prometheus, Grafana og Node Exporter opsætning  
  [https://medium.com/@cryptocapchik/grafana-prometheus-node-exporter-setup-guide-5371bef49ce0](https://medium.com/@cryptocapchik/grafana-prometheus-node-exporter-setup-guide-5371bef49ce0)

## Dokumentation

- **MkDocs**  
  Værktøj til dokumentation.  
  [https://www.mkdocs.org/](https://www.mkdocs.org/)

- **Material for MkDocs**  
  Tema til flot og funktionel dokumentationsside.  
  [https://squidfunk.github.io/mkdocs-material/](https://squidfunk.github.io/mkdocs-material/)

## Scripting og sikkerhed

- **PowerShell**  
  Bruges til firewall automation og lockdown-mode på Windows-serveren.  
  [https://learn.microsoft.com/en-us/powershell/](https://learn.microsoft.com/en-us/powershell/)

- **iptables**  
  Firewallstyring i Linux, brugt til reverse proxies.  
  [https://linux.die.net/man/8/iptables](https://linux.die.net/man/8/iptables)

---

Denne kildeoversigt refererer til både labmiljøet og det real-world incident response-setup, og bruges aktivt i både planlægning, udvikling og evaluering.
