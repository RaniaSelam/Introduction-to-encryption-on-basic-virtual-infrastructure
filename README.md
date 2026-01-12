<h1 align="center">🔐 Secure Infrastructure Lab</h1>

<p align="center">
  <i>A simple 3-machine virtual lab — built to practice defense in depth, one layer at a time.</i>
</p>

<p align="center">
  <img src="https://media.giphy.com/media/077i6AULCXc0FKTj9s/giphy.gif" width="200" alt="lock gif">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/built%20with-VirtualBox-183A61?style=flat-square&logo=virtualbox&logoColor=white"/>
  <img src="https://img.shields.io/badge/OS-Ubuntu%2022.04-E95420?style=flat-square&logo=ubuntu&logoColor=white"/>
  <img src="https://img.shields.io/badge/DB-PostgreSQL-336791?style=flat-square&logo=postgresql&logoColor=white"/>
  <img src="https://img.shields.io/badge/made%20with-curiosity%20%26%20stubbornness-ff6b6b?style=flat-square"/>
</p>

---

<h2>👋 What is this?</h2>

<p>
This is a hands-on lab I built to practice real-world cybersecurity concepts in a controlled environment. Nothing fancy — just three virtual machines, each with a strict role, connected through a private network that no one can reach from the outside.

The goal wasn't to build something complex. It was to build something <b>correct</b> — where every security decision has a reason, every layer compensates for the limits of the one before it, and compromising one machine doesn't mean game over.

On top of the infrastructure, I also deployed a small AI-assisted incident qualification system, because why not put the whole thing to use 🙂
</p>

---

<h2>🏗️ The infrastructure at a glance</h2>

<p>Three machines, each with one job:</p>

<table>
  <thead>
    <tr>
      <th>Machine</th>
      <th>IP</th>
      <th>Role</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>🛡️ <b>Bastion</b> (<code>rano</code>)</td>
      <td><code>192.168.56.20</code></td>
      <td>The only entry point — SSH hardened + fail2ban</td>
    </tr>
    <tr>
      <td>⚙️ <b>App Server</b> (<code>momo</code>)</td>
      <td><code>192.168.56.30</code></td>
      <td>Hosts the Flask API and the AI service</td>
    </tr>
    <tr>
      <td>🗄️ <b>Database</b> (<code>zizou</code>)</td>
      <td><code>192.168.56.40</code></td>
      <td>PostgreSQL — most isolated machine, encrypted disk</td>
    </tr>
  </tbody>
</table>

<p>
The bastion is the <b>only machine with two network interfaces</b> — one facing outward (NAT), one on the internal network (host-only). The app and database servers only have the internal interface and are never reachable directly from outside.
</p>

```
[You]
  │
  ▼  SSH
[Bastion] ──► [App Server] ──► [Database]
192.168.56.20   192.168.56.30   192.168.56.40
```

---

<h2>📦 What's in this repo</h2>

```
secure-infra-lab/
├── tp1-infrastructure/      # Segmented network + SSH hardening + UFW firewall rules
├── tp2-luks-encryption/     # LUKS disk encryption on the database server
└── tp3-ai-soc-assistant/    # Expert system + AI layer for incident qualification
```

<p>Each part builds on the previous one. You can explore them independently, but they're designed to work together.</p>

---

<h2>🔑 TP1 — Network segmentation & SSH hardening</h2>

<p align="center">
  <img src="https://media.giphy.com/media/3oKIPnAiaMCws8nOsE/giphy.gif" width="180" alt="network gif">
</p>

<p>The basics — but done properly.</p>

<ul>
  <li><b>ED25519 keys only</b> — no passwords, no brute-force possible</li>
  <li><b>Distinct key pairs per relationship</b> — bastion→app and app→db each have their own key. If one is stolen, the rest of the chain stays safe.</li>
  <li><b>Root login disabled</b> everywhere</li>
  <li><b>UFW rules per machine</b> — each machine only accepts what it strictly needs. The database accepts connections from the app server only. Nothing else.</li>
  <li><b>Fail2ban on the bastion</b> — the only machine exposed to the outside. It auto-bans IPs that generate too many failed login attempts.</li>
  <li><b>TLS 1.3</b> between the app server and the database — traffic is encrypted in transit even on the internal network.</li>
</ul>

> **Why TLS on an internal network?** VirtualBox's host-only network isn't encrypted. If someone compromises a VM and sniffs traffic, they see everything in plaintext without TLS. Defense in depth means not trusting the network even when it's yours.

---

<h2>🔒 TP2 — LUKS disk encryption</h2>

<p align="center">
  <img src="https://media.giphy.com/media/l0HlBO7eyXzSZkJri/giphy.gif" width="180" alt="encryption gif">
</p>

<p>
The threat scenario: someone copies the <code>.vdi</code> file (the VM's virtual disk) and analyzes it offline. Without encryption, every record in the database is readable. With LUKS, it's completely unreadable without the passphrase — even with direct access to the raw disk.
</p>

<p>
Rather than encrypting the entire system disk (complex, risky on an existing VM), a <b>dedicated secondary disk</b> was added to the database VM and encrypted with LUKS. This disk stores only the PostgreSQL data directory — clean, targeted, easy to audit.
</p>

<h4>Cryptographic parameters</h4>

```
Format:   LUKS2
Cipher:   AES-XTS-plain64  (512-bit key)
PBKDF:    Argon2id
```

**Argon2id** is a memory-hard key derivation function — intentionally slow and RAM-intensive, which makes dictionary attacks very costly even with dedicated hardware.

<h4>What LUKS protects (and what it doesn't)</h4>

<table>
  <tr>
    <td>✅ <b>Protects against</b></td>
    <td>Theft of the VM, offline disk analysis, physical access to storage</td>
  </tr>
  <tr>
    <td>❌ <b>Does NOT protect against</b></td>
    <td>Network attacks, app vulnerabilities, or a legitimate user with elevated privileges</td>
  </tr>
</table>

<p>LUKS is one layer — not the whole answer.</p>

---

<h2>🤖 TP3 — AI-assisted SOC incident qualification</h2>

<p align="center">
  <img src="https://media.giphy.com/media/xT9IgzoKnwFNmISR8I/giphy.gif" width="180" alt="AI gif">
</p>

<p>
This is where the infrastructure gets put to work. An incident qualification system is deployed on top of it — designed to assist a SOC analyst, not replace them.
</p>

<h4>The key design decision — why not just use an LLM?</h4>

<p>
In cybersecurity, decisions need to be <b>traceable, auditable, and explainable</b>. A standalone LLM is a black box — you can't explain why it classified something as critical, and it can hallucinate. So the architecture splits responsibilities:
</p>

<table>
  <tr>
    <td>🧠 <b>Decision</b></td>
    <td>Deterministic expert system — rule-based, fully auditable, stored in PostgreSQL</td>
  </tr>
  <tr>
    <td>💬 <b>Explanation</b></td>
    <td>AI layer — explains the decision, proposes a checklist, asks follow-up questions</td>
  </tr>
</table>

<p><b>The AI never touches the decision.</b> It only receives the output from the expert system and makes it human-readable.</p>

<h4>How the pipeline works</h4>

```
[User describes an incident]
         │
         ▼
[Flask API — port 8080, via bastion only]
         │
         ├──► [Expert system — PostgreSQL JSONB rules]
         │           └──► severity + action  (fully deterministic)
         │
         └──► [AI service — port 9000, localhost only, never exposed]
                     └──► explanation + checklist + follow-up questions
```

<p>32 rules cover the most common incident types. If multiple rules match, the most severe wins — conservative by design. If none match, the incident is flagged out-of-scope with no arbitrary decision made.</p>

---

<h2>🧱 The full picture</h2>

<p>Each layer covers a different threat. Together, they mean that getting through one component isn't enough to reach the data.</p>

<table>
  <thead>
    <tr><th>Layer</th><th>Tool</th><th>Protects against</th></tr>
  </thead>
  <tbody>
    <tr><td>Access control</td><td>Bastion + SSH keys</td><td>Unauthorized admin access</td></tr>
    <tr><td>Network isolation</td><td>UFW per machine</td><td>Lateral movement between components</td></tr>
    <tr><td>Encryption in transit</td><td>TLS 1.3</td><td>Traffic sniffing between app and DB</td></tr>
    <tr><td>Encryption at rest</td><td>LUKS2 + AES-XTS</td><td>Physical theft of storage</td></tr>
    <tr><td>Intrusion detection</td><td>Fail2ban</td><td>SSH brute-force on the entry point</td></tr>
    <tr><td>Traceability</td><td>History table + logs</td><td>Post-incident analysis and audit</td></tr>
  </tbody>
</table>

---

<h2>🛠️ Stack</h2>

<p>
<code>VirtualBox</code> · <code>Ubuntu 22.04</code> · <code>PostgreSQL 14</code> · <code>Flask</code> · <code>UFW</code> · <code>fail2ban</code> · <code>LUKS2</code> · <code>cryptsetup</code> · <code>Nginx</code> · <code>OpenSSH</code> · <code>ED25519</code>
</p>

---

<p align="center"><i>Built at Guardia Cybersecurity School ✨</i></p>
