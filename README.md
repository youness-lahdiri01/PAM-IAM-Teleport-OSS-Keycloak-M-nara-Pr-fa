# PAM & IAM — Teleport OSS + Keycloak @ Ménara Préfa

Privileged Access Management (PAM) and Identity & Access Management (IAM) lab that replaces direct SSH/RDP administration with a single audited bastion, role-based access control, MFA, database access control, SIEM correlation, and federated identity.

Built as a graduation project (PFA, Cybersecurity engineering track) for **Ménara Préfa**, a Moroccan precast-concrete manufacturer.

> Full write-up: `Rapport_PFA_LAHDIRI_PAM_Teleport_MenaraPrefa.pdf`
<img width="1536" height="1024" alt="ChatGPT Image Jul 26, 2026, 06_48_11 PM" src="https://github.com/user-attachments/assets/570eae20-a3e7-4300-8d89-3ed5d34c6867" />

## Why

Before this project, admins reached production servers directly over SSH and RDP: shared accounts, static passwords that were never rotated, zero command auditing, and the LAN wide open. No individual accountability, no incident traceability.

This project centralizes every privileged access behind [Teleport OSS](https://goteleport.com/) and layers [Keycloak](https://www.keycloak.org/) on top for federated identity — closing the four gaps below.

| Before | After |
|---|---|
| Direct SSH/RDP, no gateway | Single bastion (Teleport Proxy) |
| Shared accounts, static passwords | Named accounts, MFA/TOTP, ephemeral certs |
| No command audit | Full session recording, replayable SSH/RDP |
| No detection | Wazuh SIEM correlation + auto-lock + email alert |

## Architecture


Administrators authenticate once against the **Teleport bastion** (`192.168.19.150`), which fronts Auth, Proxy, and Windows Desktop services. RBAC decides what each identity can reach; every session is recorded and shipped to Wazuh.

- **PostgreSQL access** is brokered through Teleport's `db_service` over mutual TLS — no static DB passwords, every query attributable to a Teleport identity.
- **Windows access** goes through the Windows Desktop Service (WDS): a full RDP session rendered in-browser (HTML5), no RDP client, no port 3389 exposed on the LAN.
- **Wazuh SIEM** consumes Teleport's audit stream and drives a lightweight SOAR: critical events (e.g. destructive commands) trigger `tctl lock` on the offending account plus an email to the security admin.

<details>
<summary>PostgreSQL access — certificate authority chain (click to expand)</summary>


</details>

## Tech stack

| Layer | Technology |
|---|---|
| PAM / bastion | Teleport OSS **v18.9.2** (Auth, Proxy, Windows Desktop, DB services) |
| IAM / federation | Keycloak (Docker) — OIDC, LDAP User Federation |
| Directory | Active Directory + AD CS (LDAPS), Windows Server 2022 |
| Database | PostgreSQL 16, mTLS via Teleport `db_service` |
| SIEM / SOC | Wazuh 4.14.6 |
| MFA | TOTP (Google/Microsoft Authenticator) |
| OS targets | Ubuntu 24.04 (Linux nodes), Windows Server 2022 |
| Virtualization | VMware Workstation, NAT network `192.168.19.0/24` |

## Lab topology

| VM | IP | OS | RAM | Role |
|---|---|---|---|---|
| `teleport-auth` | 192.168.19.150 | Ubuntu 24.04 | 2 GB | Bastion — Auth + Proxy + WDS + Keycloak (Docker) |
| `teleport-node` | 192.168.19.155 | Ubuntu 24.04 | 1 GB | Production Linux — Apache, PostgreSQL, VSFTPD |
| `WIN-83QO4BET3NO` | 192.168.19.162 | Windows Server 2022 | 4 GB | Domain Controller — AD, DNS, AD CS |

Resources are labeled for granular RBAC targeting:

```yaml
# teleport-auth
environment: security
owner: security-team
role: teleport-gateway

# teleport-node
apache: "true"
database: "true"
ftp: "true"
environment: production
owner: infrastructure
platform: linux

# WIN-83QO4BET3NO
role: windows
environment: production
os: windows
```

## RBAC model — 9 roles

<img width="2640" height="1460" alt="rbac_model" src="https://github.com/user-attachments/assets/457c366b-e8a0-4cb0-a129-32280ce46ed8" />

Six custom, service-scoped roles enforce least privilege; three Teleport built-ins are attached to the global admin account.

| Role | System login | Access |
|---|---|---|
| `web-admin` | `webadmin` | Nodes labeled `apache: "true"` |
| `db-admin` | `dbadmin` | Nodes labeled `database: "true"` |
| `ftp-admin` | `ftpadmin` | Nodes labeled `ftp: "true"` |
| `helpdesk` | — | All Windows desktops (RDP only, zero Linux access) |
| `devops` | `ubuntu` | All Linux nodes via SSH (zero Windows access) |
| `security-admin` | — | Cluster admin only — roles, users, tokens, sessions, audit events. No production resource access. |
| `access`, `editor`, `auditor` | — | Teleport built-ins, attached to the global `admin` account |

Security Administrator is deliberately walled off from Linux and Windows production resources — separation of duties between "who can touch the bastion" and "who can touch the servers."

## Quick start

### 1. Install Teleport OSS on the bastion

```bash
# Add the official APT repo (stable v18) and install
curl https://goteleport.com/static/install.sh | bash -s v18.9.2

sudo systemctl enable teleport
sudo systemctl start teleport
```

### 2. Issue a custom TLS certificate

```bash
openssl req -x509 -newkey rsa:4096 -keyout teleport.key -out teleport.crt \
  -days 365 -nodes \
  -subj "/CN=192.168.19.150" \
  -addext "subjectAltName=IP:192.168.19.150,DNS:teleport-auth"
```

### 3. Create the global admin account (MFA-enforced)

```bash
sudo tctl users add admin --roles=editor,access,auditor --logins=root,ubuntu
# Open the invite link, set the password, enroll TOTP
```

### 4. Enroll a Linux node

```bash
# On the bastion — generate a 1-hour enrollment token
sudo tctl nodes add --ttl=1h

# On teleport-node
sudo teleport node configure --proxy=192.168.19.150:3080 --token=<TOKEN> \
  --labels=apache=true,database=true,ftp=true,environment=production,platform=linux
sudo systemctl restart teleport
```

### 5. Define and load the RBAC roles

```bash
# roles/db-admin.yaml
kind: role
version: v7
metadata:
  name: db-admin
spec:
  allow:
    logins: [dbadmin]
    node_labels:
      database: ["true"]
```

```bash
sudo tctl create -f roles/*.yaml
sudo tctl get roles --format=text
```

### 6. Create named users and enroll MFA

```bash
sudo tctl users add devops1 --roles=devops --logins=ubuntu
sudo tctl users add helpdesk1 --roles=helpdesk
```

### 7. Integrate PostgreSQL (mTLS, no static passwords)

```bash
# /etc/teleport.yaml — enable db_service
db_service:
  enabled: true
  databases:
  - name: menara-postgres-prod
    protocol: postgres
    uri: 192.168.19.155:5432
    tls:
      mode: verify-full          # not "insecure" — see Known limitations
      ca_cert_file: /etc/teleport/menaraprefa-postgres-ca.pem

# Connect
tsh db login menara-postgres-prod --db-user=menara_admin --db-name=menara_prefa_db
tsh db connect menara-postgres-prod
```

### 8. Deploy Keycloak and federate Active Directory

```bash
docker run -d --name keycloak -p 8090:8080 \
  -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=<change-me> \
  quay.io/keycloak/keycloak:latest start-dev

# In the Keycloak console: create realm `menara-prefa`,
# add a User Federation provider of type LDAP pointing at
# WIN-83QO4BET3NO.pfa-lab.local:636 (LDAPS)
```

### 9. Wire Wazuh SIEM to the bastion

```bash
sudo curl -so wazuh-agent.deb https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.6-1_amd64.deb
sudo WAZUH_MANAGER='<manager-ip>' dpkg -i ./wazuh-agent.deb
sudo systemctl enable --now wazuh-agent
```

Custom rules correlate Teleport audit events (`db.session.start`, destructive shell commands) to trigger `tctl lock <user>` and an SMTP alert — see `wazuh/local_rules.xml` (not included in this repo; see report §6.4).

## Validation scenarios

| Scenario | Test | Result |
|---|---|---|
| Stolen credentials | Correct password, missing/wrong TOTP | Login rejected at the MFA step |
| Unauthorized command | Non-privileged account runs `useradd hacker` | Blocked — `useradd` binary outside a standard user's `PATH`; RBAC + OS least-privilege both hold |
| Accidental deletion | Admin deletes a production config file by mistake | Session replayed, destructive command identified, file restored from backup |

```bash
# reproduce the lock scenario
sudo tctl lock --user=<suspect> --message="compromise suspected" --ttl=1h
sudo tctl get lock
tsh ssh ubuntu@teleport-node   # as the locked user → explicit rejection
```

## Results

| Metric | Value |
|---|---|
| Administrators under MFA | 7 |
| RBAC roles | 9 (6 custom + 3 built-in) |
| RBAC violations blocked in testing | 4 |
| Audited events (UTC timestamped) | 87+ |
| Recorded sessions | 18 SSH · 9 RDP |
| AD accounts federated into Keycloak | 7 (zero manual creation) |

**Risk before → after**

| Threat | Before | After |
|---|---|---|
| Password theft | High | Low — MFA/TOTP mandatory |
| Insider misuse | High | Low — full session recording & audit |
| Unauthorized RDP access | High | Low — RBAC on `windows_desktop_labels` |
| Shared/generic accounts | High | Low — named accounts, per-user RBAC |
| Lack of accountability | High | Low — full audit trail |
| Lateral movement | High | Medium — short-TTL ephemeral certs |
| Privilege escalation | High | Low — RBAC + OS least privilege (defense in depth) |

**Compliance coverage**

- **ISO 27001** — A.9.4.2, A.9.4.4, A.12.4.1, A.9.2.3
- **NIST SP 800-53** — AC-2, AC-6, IA-2
- **CIS Controls v8** — Control 5, Control 8

## Known limitations & roadmap

- `db_service.tls.mode` should be `verify-full` against the generated `MenaraPrefa-Postgres-CA`, not `insecure` — the one place the current lab config doesn't match the documented architecture.
- Account lock (`tctl lock`) currently expires after 1h with no follow-up notification beyond the initial email.
- No empirical before/after `nmap` scan documenting the LAN attack-surface reduction (ports 22/3389 closed, only 3080 exposed).
- Wazuh rule coverage is limited to `rm`, `chmod`, `useradd`, `systemctl` — `scp`/`wget`/`curl` (exfiltration), `history -c` (anti-forensics), and off-hours logins are not yet covered.
- Access is standing/permanent — no Just-In-Time access requests (`tsh login --request-roles=... --reason=...`) yet, despite being available in Teleport OSS.
- RBAC role creation is manual via `tctl` — no Terraform/Ansible automation.
- Keycloak groups (from the AD federation) are not yet mapped to Teleport RBAC roles — the two identity layers work side by side rather than converged.
- Next: pfSense VLAN segmentation, preventive command validation (approve-before-execute instead of block-after-detect), continued alignment with MITRE ATT&CK.

## Repository structure

```
.
├── README.md
├── readme_assets/            # architecture & RBAC diagrams
├── roles/                    # Teleport RBAC role definitions (YAML)
├── teleport/                 # teleport.yaml, systemd units, TLS certs (gitignored)
├── keycloak/                 # realm export, LDAP federation config
└── wazuh/                    # local_rules.xml, ossec.conf snippets
```

## Author

**LAHDIRI Youness** — Cybersecurity engineering, EMSI (4th year)
Graduation project (PFA) hosted at **Ménara Préfa**
Supervised by M. MRHIOUNI Amine (Ménara Préfa)

## License

Educational project — no license granted for production reuse without permission.
