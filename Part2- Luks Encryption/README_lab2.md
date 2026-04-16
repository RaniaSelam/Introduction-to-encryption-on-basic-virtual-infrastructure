<h1 align="center">🔒 Lab 2 — LUKS Disk Encryption</h1>

<p align="center">
  <i>Because network security alone isn't enough if someone can just walk away with the disk.🚶🏻‍♂️‍➡️🚶🏻‍♂️‍➡️</i>
</p>

<h2>🎯 Goal</h2>

<p>
Add a layer of <b>data-at-rest protection</b> to the infrastructure built in Lab 1. The target is the database server — identified as the most critical asset — using LUKS disk encryption on a dedicated secondary disk.
</p>

<h2> Risk analysis first</h2>

<p>Before any technical implementation, a risk analysis has to be done to identify which component actually needs encryption.</p>

<p>The main threat scenario considered:</p>
<ul>
  <li>theft or loss of a virtual machine</li>
  <li>unauthorized access to the virtual disk from the hypervisor</li>
  <li>offline analysis of disk content</li>
</ul>

<p>In this scenario, all the network security mechanisms from Lab 1 (firewall, SSH, TLS) become <b>completely ineffective</b> — the attacker isn't attacking a running system, they're reading the raw storage directly.</p>

<h4>Why the database and not the other machines?</h4>

<table>
  <thead>
    <tr><th>Machine</th><th>What it contains</th><th>If stolen</th></tr>
  </thead>
  <tbody>
    <tr>
      <td> Bastion</td>
      <td>SSH config, logs</td>
      <td>Low data impact — reconstructible</td>
    </tr>
    <tr>
      <td> App Server</td>
      <td>Code, config files</td>
      <td>Reconstructible from source</td>
    </tr>
    <tr>
      <td> <b>Database</b></td>
      <td><b>Persistent data, potentially sensitive</b></td>
      <td><b>Irreversible — data is gone</b></td>
    </tr>
  </tbody>
</table>

<p>The database was the clear priority. The bastion was also used as a test machine to validate the approach before applying it to the most critical component.</p>

<h2> Architecture choice — selective encryption</h2>

<p>Three approaches were considered:</p>

<ul>
  <li><b>Full system disk encryption</b> — maximum protection, but heavy to set up on an existing VM and complex to recover from</li>
  <li><b>Selective encryption on a dedicated partition</b> ← chosen approach</li>
  <li><b>Hybrid</b> — mix of encrypted and non-encrypted partitions</li>
</ul>

<p>A <b>secondary disk was added to the database VM</b> and encrypted with LUKS. This disk stores only the PostgreSQL data directory (<code>/var/lib/postgresql</code>).</p>

<p>Benefits of this approach:</p>
<ul>
  <li>No modification to the existing OS</li>
  <li>Limited blast radius if something goes wrong during setup</li>
  <li>Easier to audit — the encryption boundary is explicit and clean</li>
</ul>

---

<h2>🔐 Cryptographic parameters</h2>

```
Format:    LUKS2
Cipher:    AES-XTS-plain64
Key size:  512 bits
PBKDF:     Argon2id
```

<p><b>Why Argon2id?</b> It's a memory-hard key derivation function, intentionally slow and RAM-intensive. Even with dedicated hardware, a dictionary attack against a passphrase becomes extremely costly.</p>

<p><b>Why AES-XTS?</b> It's the standard mode for disk encryption : designed specifically to handle the structure of block storage, resistant to pattern analysis across sectors.</p>

---

<h2>⚙️ How it works at runtime</h2>

<p>
LUKS sits between the physical disk and the filesystem. When the VM starts, an administrator provides the passphrase. Argon2id derives the encryption key, the kernel exposes a decrypted virtual device under <code>/dev/mapper/luks_db</code>, and PostgreSQL reads and writes to it normally, completely unaware there's encryption underneath.
</p>


<h2>🗝️ Key management</h2>

<ul>
  <li><b>Two LUKS key slots configured</b> — a main passphrase for daily use, and a recovery key in case the main one is lost</li>
  <li><b>LUKS header backed up externally</b> — the header contains all cryptographic metadata. If it's corrupted or lost, the data becomes permanently inaccessible even with the correct passphrase. Its backup is critical.</li>
  <li><b>Manual decryption at startup</b> — intentional. The volume stays locked after every reboot until an administrator explicitly unlocks it.</li>
</ul>

<h4>Startup procedure after reboot</h4>

```bash
sudo cryptsetup open /dev/sdb luks_db
sudo mount /dev/mapper/luks_db /var/lib/postgresql
sudo systemctl start postgresql
```

> This is a deliberate operational constraint — maximizing physical protection at the cost of a manual step. In production, automation via TPM or network key could be evaluated, but with a full risk analysis first, since automation reduces physical protection.

---

<h2> What LUKS protects.. and what it doesn't</h2>

<table>
  <tr>
    <td> <b>Protects against</b></td>
    <td>Theft of the VM or virtual disk file · Offline disk analysis · Physical access to storage media</td>
  </tr>
  <tr>
    <td> <b>! Does NOT protect against !</b></td>
    <td>Network attacks · Application vulnerabilities · A legitimate user with elevated privileges · Any attack while the volume is already mounted</td>
  </tr>
</table>

<p>LUKS is one layer in a defense-in-depth strategy — not a complete solution on its own. Once the volume is mounted and the system is running, data flows normally. The other layers from Lab 1 (bastion, firewall, TLS) cover the threats LUKS doesn't.</p>



<p align="center"><i>← <a href="../1-%20Infrastructure/README.md">Lab 1 — Infrastructure</a> · <a href="../README.md">Back to main README →</a></i></p>
