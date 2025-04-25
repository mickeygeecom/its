# Firewall-manager (Windows Server)

Denne lille Node-service giver et REST-lag omkring **Windows Firewall‐cmdletten**
(*New-NetFirewallRule* / *Remove-NetFirewallRule*).  
Den er udviklet til hurtigt at **whiteliste FiveM-spillere** på port **30121**  
(både UDP og TCP) – men kan genbruges til andre porte.

| End-point        | Beskrivelse                                   |
|------------------|-----------------------------------------------|
| `/wl?ip=<IP>`    | Opretter to regler (UDP + TCP) for den IP.    |
| `/unwl?ip=<IP>`  | Fjerner begge regler for den IP.              |
| `/unwlall`       | Fjerner **alle** regler i firewall-gruppen `FIVEM`. |

!!! note "Forudsætninger"
    * Windows Server 2019/2022 eller Windows 10/11 med PowerShell 5+
    * Node.js ≥ 18
    * Admin-rettigheder (krævet for at oprette firewall‐regler)

---

## Kildekode – `server.js`

```javascript title="server.js" linenums
const express  = require('express');
const { exec } = require('child_process');
const app      = express();

const port = process.env.PORT || 3000;

/* ---------- Helper: opret firewall-regel ---------- */
function addRule(ip, cb) {
  const udp = `FIVEM_WHITELIST_${ip}_UDP`;
  const tcp = `FIVEM_WHITELIST_${ip}_TCP`;

  const cmd = `powershell.exe -Command `
    + `"if (Get-NetFirewallRule -DisplayName '${udp}' -ErrorAction SilentlyContinue) `
    + `{ 'UDP Rule already exists' } else { `
    + `New-NetFirewallRule -DisplayName '${udp}' -Direction Inbound `
    + `-Action Allow -Protocol UDP -RemoteAddress ${ip} -LocalPort 30121 -Group 'FIVEM' }; `
    + `if (Get-NetFirewallRule -DisplayName '${tcp}' -ErrorAction SilentlyContinue) `
    + `{ 'TCP Rule already exists' } else { `
    + `New-NetFirewallRule -DisplayName '${tcp}' -Direction Inbound `
    + `-Action Allow -Protocol TCP -RemoteAddress ${ip} -LocalPort 30121 -Group 'FIVEM' }"`;

  exec(cmd, (err, out, stderr) => err ? cb(err, stderr) : cb(null, out));
}

/* ---------- Helper: fjern firewall-regel for IP ---------- */
function removeRule(ip, cb) {
  const base = `FIVEM_WHITELIST_${ip}`;
  const cmd  = `powershell.exe -Command `
             + `"if (Get-NetFirewallRule -DisplayName '${base}'* -ErrorAction SilentlyContinue) `
             + `{ Remove-NetFirewallRule -DisplayName '${base}'* } `
             + `else { 'Rule not found' }"`;

  exec(cmd, (err, out, stderr) => err ? cb(err, stderr) : cb(null, out));
}

/* ---------- Helper: fjern alle regler i gruppen ---------- */
function removeAllRules(cb) {
  const cmd = `powershell.exe -Command `
            + `"if (Get-NetFirewallRule -Group 'FIVEM' -ErrorAction SilentlyContinue) `
            + `{ Remove-NetFirewallRule -Group 'FIVEM' } `
            + `else { 'No rules found in group FIVEM' }"`;

  exec(cmd, (err, out, stderr) => err ? cb(err, stderr) : cb(null, out));
}

/* ---------- REST-routes ---------- */
app.get('/wl',      (req, res) => !req.query.ip ? res.status(400).send('IP parameter is required')
                                                : addRule(req.query.ip,   rsp(res)));
app.get('/unwl',    (req, res) => !req.query.ip ? res.status(400).send('IP parameter is required')
                                                : removeRule(req.query.ip, rsp(res)));
app.get('/unwlall', (req, res) => removeAllRules(rsp(res)));

function rsp(res) {
  return (err, out) => err ? res.status(500).send(`Error: ${out}`) : res.send('ok');
}

app.listen(port, '0.0.0.0', () =>
  console.log(`Firewall-manager kører på port ${port}`)
);
