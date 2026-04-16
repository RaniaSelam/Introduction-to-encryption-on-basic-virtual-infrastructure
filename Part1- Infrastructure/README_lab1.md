<h1 align="center">🛡️ Lab 1 — Secure Virtualized Infrastructure</h1>

<p align="center">
  <i>Three machines, one entry point, zero unnecessary exposure.</i>
</p>

<h2>🎯 Goal</h2>

<p>
Design and deploy a segmented 3-machine infrastructure, where each machine has a strict role and can only communicate with what it strictly needs. The focus is on <b>network architecture, SSH hardening, firewall rules, and encrypted communications</b> — not on application complexity.
</p>


<h2> Architecture</h2>

<p>All three machines run <b>Ubuntu Server 22.04 LTS</b> and are connected via a VirtualBox host-only network. The bastion is the only machine with two interfaces — one NAT for internet access, one host-only for the internal network. The app and database servers only have the internal interface and are never directly reachable from outside.</p>

<table>
  <thead>
    <tr>
      <th>Machine</th>
      <th>Hostname</th>
      <th>User</th>
      <th>IP (host-only)</th>
      <th>Role</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td> <b>Bastion</b></td>
      <td><code>bastion</code></td>
      <td><code>rano</code></td>
      <td><code>192.168.56.20</code></td>
      <td>Single entry point — SSH + fail2ban</td>
    </tr>
    <tr>
      <td> <b>App Server</b></td>
      <td><code>app</code></td>
      <td><code>momo</code></td>
      <td><code>192.168.56.30</code></td>
      <td>Nginx + Flask API</td>
    </tr>
    <tr>
      <td> <b>Database</b></td>
      <td><code>db</code></td>
      <td><code>zizou</code></td>
      <td><code>192.168.56.40</code></td>
      <td>PostgreSQL 14 — most isolated machine</td>
    </tr>
  </tbody>
</table>

<h4>Resources allocated per machine</h4>

<table>
  <thead>
    <tr><th>Machine</th><th>CPU</th><th>RAM</th><th>Disk</th></tr>
  </thead>
  <tbody>
    <tr><td>Bastion</td><td>1 vCPU</td><td>2 GB</td><td>~20 GB</td></tr>
    <tr><td>App</td><td>1 vCPU</td><td>2 GB</td><td>~20 GB</td></tr>
    <tr><td>DB</td><td>1 vCPU</td><td>2 GB</td><td>~20 GB</td></tr>
  </tbody>
</table>

<h4>Deployment approach</h4>

<p>
The bastion was set up first, then <b>cloned twice</b> to create the app and database servers. Each clone was then individually reconfigured: hostname, dedicated user, network interfaces (NAT removed on internal machines), and netplan config. This approach guaranteed a consistent base OS across all machines while saving deployment time.
</p>


<h2> SSH hardening</h2>


<p>All administrative access goes through SSH. Password authentication was disabled from the start — no exceptions.</p>

<ul>
  <li><b>ED25519 keys only</b> — stronger and faster than RSA, smaller key size</li>
  <li><b>PasswordAuthentication no</b> — no brute-force possible</li>
  <li><b>PermitRootLogin no</b> — no direct root access</li>
  <li><b>AllowUsers</b> — only the dedicated user can connect on each machine</li>
  <li><b>Strict permissions</b> — <code>~/.ssh</code> set to 700, <code>authorized_keys</code> to 600</li>
</ul>

<h4>Distinct key pairs per relationship</h4>

<p>Three separate key pairs were generated — one per relationship:</p>

```
Admin workstation ──(key_bastion)──► Bastion
Bastion           ──(key_app)──────► App Server
App Server        ──(key_db)───────► Database
```

<p>If one key is compromised, only one link in the chain is exposed. The rest stays intact.</p>

> **Note:** Password authentication was disabled before taking screenshots — re-enabling it just for a screenshot would have been a security regression. The result: `Permission denied (publickey)` on password attempts, successful login with the correct key. That's exactly the expected behavior.

---

<h2> Firewall (UFW) — per-machine rules</h2>

<p>Default policy on every machine: <b>deny incoming, allow outgoing</b>. Only strictly necessary flows are opened, adapted to each machine's role.</p>

<h4>Bastion</h4>
<ul>
  <li>SSH incoming from the admin workstation only (<code>192.168.56.10</code>)</li>
  <li>SSH outgoing to the app server</li>
  <li>Everything else blocked</li>
  <li>IPv6 disabled — unused, reduces attack surface</li>
</ul>

<h4>App Server</h4>
<ul>
  <li>SSH incoming from the bastion only (<code>192.168.56.20</code>)</li>
  <li>HTTP port 80 incoming from the bastion</li>
  <li>Outgoing connections to the database</li>
</ul>

<h4>Database</h4>
<ul>
  <li>PostgreSQL port 5432 from the app server only (<code>192.168.56.30</code>)</li>
  <li>SSH from the app server only</li>
  <li>Nothing else — most isolated machine in the infrastructure</li>
</ul>

> **Why no direct SSH from bastion to DB?** The database is the most critical machine. Forcing the path bastion → app → db adds one more barrier. An attacker who compromises the bastion still can't reach the database directly.



<h2>🚫 Fail2ban</h2>

<p>
Installed <b>only on the bastion</b> — it's the only machine exposed to the outside. Fail2ban monitors SSH logs and automatically bans IPs that generate too many failed login attempts.
</p>

<p>Even with password auth disabled, it limits noise and random key attempts. The SSH jail was verified active with <code>fail2ban-client status sshd</code>.</p>



<h2> TLS 1.3 — encryption in transit</h2>

<p>
Communications between the app server and the database are encrypted with <b>TLS 1.3</b>. PostgreSQL was chosen over MySQL specifically for its <b>native TLS support</b> — no external configuration needed.
</p>

<p>A dedicated application user <code>appuser</code> was created with only the rights needed — no superuser access.</p>

> **Why TLS on an internal network?** VirtualBox's host-only network isn't encrypted. If someone compromises a VM and sniffs traffic, everything is visible in plaintext without TLS. Defense in depth means not trusting the network even when it's yours.



<h2> Challenges encountered</h2>

<p><b>PostgreSQL not reachable from the network after cluster recreation</b> — by default PostgreSQL only listens on <code>127.0.0.1</code>. Fixed by updating <code>listen_addresses</code> in <code>postgresql.conf</code> and adding a rule in <code>pg_hba.conf</code> to allow the app server's IP.</p>

<p><b>VM cloning and reconfiguration</b> — after cloning, each machine needed its hostname, user, network interfaces, and netplan config individually adjusted. The NAT interface had to be removed from the internal machines.</p>



<p align="center"><i>← <a href="../README.md">Back to main README</a> · <a href="../2-%20Luks%20Encryption/README.md">Lab 2 — LUKS Encryption →</a></i></p>
