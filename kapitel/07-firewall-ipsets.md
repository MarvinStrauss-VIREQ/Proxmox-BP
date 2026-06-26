<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: PVE – Best Practices -->
<!-- Title: PVE – Sicherheit: Firewall & IPsets -->
<!-- Label: pve-sicherheit, best-practice, referenz, pve-cluster -->

# Kapitel 07: Firewall & IPsets

> **PVE 9.x (Debian 13 Trixie)** · Letzte Aktualisierung: 2026-06  
> Verwandte Kapitel: [02 – Netzwerk & Clustering](02-netzwerk-clustering.md) · [05 – Sicherheit & Hardening](05-sicherheit-hardening.md)

---

## Wann anwenden?

- **Jede Neuinstallation** – Firewall aktivieren bevor der Node produktiv geht
- **3-Node-Cluster mit Ceph** – Spezielles Regelset für Monitor-, OSD- und MGR-Ports
- **2-Node-Cluster** – Vereinfachtes Regelset ohne Ceph-Regeln (Abschnitt unten)
- **Netzwerksegmentierung** – Management-, Storage- und VM-Traffic auf Hypervisor-Ebene trennen
- **Sicherheits-Audit** – Bestehende Firewall-Config gegen VIREQ-Standard prüfen

> **Nicht anwenden**, wenn: akuter Troubleshooting-Kontext mit laufenden Ausfällen → erst Problem lösen, dann Firewall aktivieren.

---

## Überblick

Die `pve-firewall` ist Proxmox-nativ und operiert auf **drei unabhängigen Ebenen**:

| Ebene | Konfigurationsdatei | Geltungsbereich |
|-------|--------------------|--------------------|
| **Datacenter** | `/etc/pve/firewall/cluster.fw` | Cluster-weit für alle Nodes (pmxcfs-repliziert) |
| **Host** | `/etc/pve/nodes/<node>/host.fw` | Node-spezifische Ergänzungen |
| **VM/CT** | `/etc/pve/firewall/<vmid>.fw` | Einzelne VMs oder Container |

### Hyper-V Vergleich

| Konzept | Hyper-V | Proxmox VE |
|---------|---------|------------|
| Host-Firewall | Windows Firewall (GPO oder lokale Regeln) | `pve-firewall` (clusternativ) |
| Zentrales Management | GPO / Windows Admin Center | Datacenter-Level (`cluster.fw`) |
| IP-Gruppen | Adressbereiche im Firewall-Scope | **IPsets** – einmal definiert, cluster-weit per pmxcfs |
| Regelgruppen für VMs | kein natives Äquivalent | **Security Groups** – benannte Regelsets, in VMs referenzierbar |
| VM-Firewall | Hyper-V Port-ACLs (vSwitch-Ebene) | VM-Level `.fw`-Dateien unter `/etc/pve/firewall/` |
| Regelverteilung | Active Directory / GPO-Replikation | pmxcfs – automatisch, ohne AD-Abhängigkeit |
| Default-Policy | Allow (muss manuell gehärtet werden) | Konfigurierbar – `DROP` ist VIREQ-Standard |

---

## Firewall-Regel-Syntax

> **Kritisch:** Diese drei Punkte unterscheiden sich von anderen Firewalls und verursachen Compile-Fehler wenn falsch angewendet.

| Parameter | Falsch ❌ | Richtig ✅ | Grund |
|-----------|----------|----------|-------|
| Kommentare | `-comment "SSH: Admin"` | `# SSH: Admin` am Zeilenende | `-comment` ist kein gültiger Parameter |
| Protokoll | `-proto tcp` | `-p tcp` | Kanonische Form (entspricht UI-Output) |
| Datacenter-IPsets | `+mgmt-hosts` | `+dc/mgmt-hosts` | Expliziter Scope – Pflicht in Security Groups, empfohlen in allen Regeln |

```ini
# So generiert die Proxmox-UI Regeln – das ist die Referenz:
IN ACCEPT -source +dc/mgmt-hosts -p tcp -dport 22 -log nolog # SSH: Admin-Zugriff
```

---

## Voraussetzungen

- Proxmox VE 9.x Cluster aufgebaut und Corosync aktiv
- IP-Adressen aller Nodes auf **allen drei** Netzwerken bekannt
- IPMI / iDRAC / physische Konsolenverbindung als Fallback vorab prüfen

> ⚠️ **Vor der Aktivierung:** Eigene Admin-IP in `mgmt-hosts` eintragen – sonst Selbst-Lock-out.

---

## NIC-Layout (6-NIC / 3-Node-Cluster mit Ceph)

```
pve1
│
├── nic0  (enp6s18)  ──→  vmbr0  ──→  172.30.7.31/16, GW 172.30.254.254
│                          └─ Web UI / Corosync Ring 1 / Admin-Zugriff
│
├── ens19 (enp6s19)  ──→  10.10.20.11/24
│                          └─ Corosync Ring 0 (dediziert)
│
├── ens20 (enp6s20)  ┐
│                    ├──→  bond0 (active-backup)  ──→  vmbr1 (VLAN-aware)
├── ens21 (enp6s21)  ┘       └─ VM-Traffic (Tagged VLANs)
│
├── ens22 (enp6s22)  ┐
│                    └──→  bond1 (balance-rr)  ──→  10.20.20.11/24
└── ens23 (enp6s23)  ┘       └─ Ceph/Storage-Netz (eigenes Subnetz, kein Gateway)
```

| Interface | Typ | IP (pve1 / pve2 / pve3) | Subnetz | Funktion |
|-----------|-----|--------------------------|---------|---------|
| `nic0` → `vmbr0` | Bridge | 172.30.7.31 / .32 / .33 | /16 | Web UI, Admin-SSH, Corosync Ring 1 |
| `ens19` | Dedizierte NIC | 10.10.20.11 / .12 / .13 | /24 | Corosync Ring 0 |
| `bond0` → `vmbr1` | Bond → Bridge | – | – | VM-Traffic, VLAN-aware |
| `bond1` | Bond (balance-rr) | 10.20.20.11 / .12 / .13 | /24 | Ceph/Storage |

---

## Das Firewall-Toolkit: IPsets + Security Groups

| Baustein | Gruppiert | Syntax | Einsatz |
|----------|-----------|--------|---------|
| **IPset** | IP-Adressen | `-source +dc/cluster-mgmt` | Quell-/Ziel-Filter |
| **Security Group** | Firewall-Regeln | `GROUP sg-web` | Regelvorlagen für VM-Typen |

---

## IPsets für den 3-Node-Cluster

| IPset-Name | Eintragsformat | Interface | Verwendung |
|------------|---------------|-----------|------------|
| `cluster-mgmt` | Einzeladressen (172.30.7.31–33) | vmbr0 | Web UI, Admin-SSH, Corosync Ring 1, Live-Migration |
| `corosync-ring0` | Einzeladressen (10.10.20.11–13) | ens19 | Corosync Ring 0 |
| `ceph-net` | Subnetz `10.20.20.0/24` | bond1 | Ceph Monitor, OSD, MDS, MGR |
| `mgmt-hosts` | Admin-IPs (variabel) | – | Admin-Workstations, Jump-Hosts |

`ceph-net` als `/24` weil bond1 physisch isoliert ist (kein Gateway) — ein fremdes Gerät kann das Subnetz nicht erreichen, und Cluster-Erweiterungen werden automatisch abgedeckt. Die anderen IPsets verwenden Einzeladressen da ihre Netze ein Gateway haben.

---

## Datacenter-Firewall konfigurieren

### Vollständige cluster.fw – 3-Node-Ceph-Cluster

```ini
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT
log_ratelimit: enable=1,burst=10,rate=5/second

########################################
# IPsets – Vor Aktivierung anpassen!
########################################

[IPSET cluster-mgmt] # Management-Netzwerk + Corosync Ring 1 (vmbr0 via nic0)
172.30.7.31 # pve1
172.30.7.32 # pve2
172.30.7.33 # pve3

[IPSET corosync-ring0] # Corosync Ring 0 – dedizierte NIC (ens19)
10.10.20.11 # pve1
10.10.20.12 # pve2
10.10.20.13 # pve3

[IPSET ceph-net] # bond1 – Ceph Storage-Netz, physisch isoliert (kein Gateway)
10.20.20.0/24

[IPSET mgmt-hosts] # Admin-Workstations / Jump-Hosts – vor Aktivierung befüllen!
# 10.0.1.50
# 10.0.1.0/24

########################################
# Security Groups (VM-Firewall-Vorlagen)
# Syntax: -p tcp|udp|icmp  /  # Kommentar am Zeilenende  /  +dc/ipset-name
########################################

[group sg-base]
IN ACCEPT -source +dc/mgmt-hosts -p tcp -dport 22 -log nolog # SSH: Admin
IN ACCEPT -p icmp -log nolog                                  # ICMP: Monitoring

[group sg-windows]
IN ACCEPT -source +dc/mgmt-hosts -p tcp -dport 3389 -log nolog # RDP: Admin
IN ACCEPT -source +dc/mgmt-hosts -p tcp -dport 5985 -log nolog # WinRM HTTP
IN ACCEPT -source +dc/mgmt-hosts -p tcp -dport 5986 -log nolog # WinRM HTTPS

[group sg-web]
IN ACCEPT -p tcp -dport 80  -log nolog # HTTP
IN ACCEPT -p tcp -dport 443 -log nolog # HTTPS

[group sg-web-internal]
IN ACCEPT -source +dc/mgmt-hosts -p tcp -dport 80  -log nolog # HTTP intern
IN ACCEPT -source +dc/mgmt-hosts -p tcp -dport 443 -log nolog # HTTPS intern

[group sg-db-mssql]
IN ACCEPT -source +dc/mgmt-hosts -p tcp -dport 1433 -log nolog # MSSQL Admin

[group sg-db-mysql]
IN ACCEPT -source +dc/mgmt-hosts -p tcp -dport 3306 -log nolog # MySQL Admin

########################################
# Host-Firewall-Regeln
########################################

[RULES]

# ── SSH ──────────────────────────────────────────────────────────────────
IN ACCEPT -source +dc/mgmt-hosts      -p tcp -dport 22    -log nolog # SSH: Admin-Zugriff
IN ACCEPT -source +dc/cluster-mgmt    -p tcp -dport 22    -log nolog # SSH: Inter-Node

# ── Proxmox Web UI ───────────────────────────────────────────────────────
IN ACCEPT -source +dc/mgmt-hosts      -p tcp -dport 8006  -log nolog # PVE Web UI

# ── Corosync Ring 0 (10.10.20.x / ens19) ────────────────────────────────
IN ACCEPT -source +dc/corosync-ring0  -p udp -dport 5405  -log nolog # Corosync Ring 0

# ── Corosync Ring 1 (172.30.7.x / vmbr0) ────────────────────────────────
IN ACCEPT -source +dc/cluster-mgmt    -p udp -dport 5406  -log nolog # Corosync Ring 1

# ── Live-Migration ───────────────────────────────────────────────────────
IN ACCEPT -source +dc/cluster-mgmt    -p tcp -dport 60000:60050 -log nolog # Live-Migration

# ── Konsole ──────────────────────────────────────────────────────────────
IN ACCEPT -source +dc/mgmt-hosts      -p tcp -dport 5900:5999 -log nolog # VNC/noVNC
IN ACCEPT -source +dc/mgmt-hosts      -p tcp -dport 3128  -log nolog # SPICE-Proxy

# ── Ceph Monitor (10.20.20.0/24 / bond1) ────────────────────────────────
IN ACCEPT -source +dc/ceph-net        -p tcp -dport 3300  -log nolog # Ceph Monitor v2
IN ACCEPT -source +dc/ceph-net        -p tcp -dport 6789  -log nolog # Ceph Monitor v1

# ── Ceph OSD / MDS / MGR ────────────────────────────────────────────────
IN ACCEPT -source +dc/ceph-net        -p tcp -dport 6800:7300 -log nolog # Ceph OSD/MDS/MGR

# ── ICMP ─────────────────────────────────────────────────────────────────
IN ACCEPT -source +dc/cluster-mgmt    -p icmp -log nolog # ICMP: Management-Netz
IN ACCEPT -source +dc/corosync-ring0  -p icmp -log nolog # ICMP: Corosync Ring 0
IN ACCEPT -source +dc/ceph-net        -p icmp -log nolog # ICMP: Ceph-Netz
IN ACCEPT -source +dc/mgmt-hosts      -p icmp -log nolog # ICMP: Admin
```

### Konfiguration einspielen

```bash
nano /etc/pve/firewall/cluster.fw
pve-firewall compile   # Dry-run – kein Output = kein Fehler
pve-firewall reload
pve-firewall status
```

---

## Security Groups – VM-Firewall mit Vorlagen

Security Groups lösen das Problem doppelter Regelsets: einmal in `cluster.fw` definieren, per `GROUP sg-name` in beliebig vielen VM-Firewalls referenzieren. Änderung an einer Stelle wirkt nach `pve-firewall reload` sofort auf alle VMs.

> Security Groups greifen **ausschließlich auf VM/CT-Ebene** — sie schützen den Hypervisor selbst nicht. Der [RULES]-Abschnitt in `cluster.fw` schützt den Hypervisor.

### Security Groups auf VMs anwenden

**Beispiel: Windows-Webserver (VM 100)**

```ini
# /etc/pve/firewall/100.fw
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT

[RULES]
GROUP sg-base      # SSH + ICMP von Admin
GROUP sg-windows   # RDP + WinRM von Admin
GROUP sg-web       # HTTP/HTTPS öffentlich
```

**Beispiel: Linux-Webserver (VM 101)**

```ini
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT

[RULES]
GROUP sg-base   # SSH + ICMP von Admin
GROUP sg-web    # HTTP/HTTPS öffentlich
```

**Beispiel: Datenbankserver (VM 102)**

```ini
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT

[RULES]
GROUP sg-base      # SSH + ICMP von Admin
GROUP sg-db-mssql  # MSSQL nur von Admin
```

```bash
# VM-Firewall aktivieren
nano /etc/pve/firewall/100.fw

# Oder Web UI: VM → Firewall → Options → Firewall: Yes
#             VM → Firewall → Add → Security Group auswählen
```

### VIREQ Standard Security Groups

| Security Group | Regeln | Typische VMs |
|---------------|--------|-------------|
| `sg-base` | SSH + ICMP (von `mgmt-hosts`) | Alle VMs — immer erste Gruppe |
| `sg-windows` | RDP 3389, WinRM 5985/5986 | Windows Server |
| `sg-web` | HTTP 80, HTTPS 443 (öffentlich) | Webserver, Reverse Proxy |
| `sg-web-internal` | HTTP/HTTPS (nur von `mgmt-hosts`) | Interne Webapps |
| `sg-db-mssql` | MSSQL 1433 (von `mgmt-hosts`) | MSSQL-Datenbankserver |
| `sg-db-mysql` | MySQL 3306 (von `mgmt-hosts`) | MySQL/MariaDB-Server |

---

## Datacenter-Firewall – 2-Node-Cluster (vereinfacht)

```ini
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT
log_ratelimit: enable=1,burst=10,rate=5/second

[IPSET cluster-mgmt]
<IP-Node1> # pve1
<IP-Node2> # pve2
# <IP-PBS>  # PBS-Node (falls als QDevice)

[IPSET mgmt-hosts]
# <Admin-IP>

[RULES]
IN ACCEPT -source +dc/mgmt-hosts    -p tcp -dport 22          -log nolog # SSH: Admin-Zugriff
IN ACCEPT -source +dc/cluster-mgmt  -p tcp -dport 22          -log nolog # SSH: Inter-Node
IN ACCEPT -source +dc/mgmt-hosts    -p tcp -dport 8006        -log nolog # PVE Web UI
IN ACCEPT -source +dc/cluster-mgmt  -p udp -dport 5405        -log nolog # Corosync Ring 0
IN ACCEPT -source +dc/cluster-mgmt  -p tcp -dport 60000:60050 -log nolog # Live-Migration
IN ACCEPT -source +dc/mgmt-hosts    -p tcp -dport 5900:5999   -log nolog # VNC/noVNC
IN ACCEPT -source +dc/mgmt-hosts    -p tcp -dport 3128        -log nolog # SPICE-Proxy
IN ACCEPT -source +dc/cluster-mgmt  -p icmp -log nolog                   # ICMP: Inter-Node
IN ACCEPT -source +dc/mgmt-hosts    -p icmp -log nolog                   # ICMP: Admin
```

---

## Host-Firewall konfigurieren

```ini
[OPTIONS]
enable: 1

[RULES]
# Prometheus Node Exporter (optional)
# IN ACCEPT -source +dc/mgmt-hosts -p tcp -dport 9100 -log nolog # Prometheus

# Ceph Dashboard (direkt, optional – normalerweise über PVE Web UI ausreichend)
# IN ACCEPT -source +dc/mgmt-hosts -p tcp -dport 8443 -log nolog # Ceph Dashboard

# NinjaOne Agent: keine eingehenden Ports
```

```bash
nano /etc/pve/nodes/pve1/host.fw
nano /etc/pve/nodes/pve2/host.fw
nano /etc/pve/nodes/pve3/host.fw
```

---

## Firewall aktivieren – Schritt für Schritt

> ⚠️ **Reihenfolge:** IPsets befüllen → Dry-run → Aktivieren → Verifizieren.

```bash
# 1. Admin-IP in mgmt-hosts eintragen
nano /etc/pve/firewall/cluster.fw

# 2. Syntaxprüfung – kein Output = kein Fehler
pve-firewall compile

# 3. Aktivieren
pvesh set /cluster/firewall/options --enable 1

# 4. Verifizieren
pve-firewall status
iptables -L INPUT -n -v --line-numbers
ipset list cluster-mgmt
ipset list corosync-ring0
ipset list ceph-net
ipset list mgmt-hosts
corosync-cfgtool -s   # Beide Ringe: "connected"
ceph status           # Kein OSD darf fallen
```

```bash
# 5. Cluster-weite Statusprüfung
for node in pve1 pve2 pve3; do
  echo "=== $node ==="
  ssh root@$node "pve-firewall status && corosync-cfgtool -s | grep -E 'ring|FAULTY'"
done
```

---

## Port-Referenz

| Port(s) | Protokoll | Dienst | Quelle (IPset) | Interface | Pflicht |
|---------|-----------|--------|---------------|-----------|:-------:|
| 22 | TCP | SSH | `mgmt-hosts`, `cluster-mgmt` | vmbr0 | ✅ |
| 8006 | TCP | Proxmox Web UI | `mgmt-hosts` | vmbr0 | ✅ |
| 5405 | UDP | Corosync Ring 0 | `corosync-ring0` | ens19 | ✅ |
| 5406 | UDP | Corosync Ring 1 | `cluster-mgmt` | vmbr0 | ✅ |
| 60000–60050 | TCP | VM/CT Live-Migration | `cluster-mgmt` | vmbr0 | ✅ |
| 5900–5999 | TCP | VNC / noVNC-Konsole | `mgmt-hosts` | vmbr0 | ✅ |
| 3128 | TCP | SPICE-Proxy | `mgmt-hosts` | vmbr0 | ✅ |
| 3300 | TCP | Ceph Monitor v2 | `ceph-net` | bond1 | ✅ (Ceph) |
| 6789 | TCP | Ceph Monitor v1 | `ceph-net` | bond1 | ✅ (Ceph) |
| 6800–7300 | TCP | Ceph OSD / MDS / MGR | `ceph-net` | bond1 | ✅ (Ceph) |
| — | ICMP | Ping / Monitoring | alle IPsets | alle | ✅ |
| 9100 | TCP | Prometheus Node Exporter | `mgmt-hosts` | vmbr0 | Optional |
| 8443 | TCP | Ceph Dashboard (direkt) | `mgmt-hosts` | vmbr0 | Optional |
| 4789 | UDP | VXLAN (SDN) | `cluster-mgmt` | vmbr0 | Optional |

---

## Best Practices

✅ **Empfohlen:**
- Kommentare immer als `# text` am Zeilenende — niemals `-comment "text"`
- IPsets immer mit `+dc/` Prefix referenzieren — auch innerhalb der `cluster.fw`
- `sg-base` immer als erste Security Group in jeder VM-Firewall einbinden
- `ceph-net` als `/24` wenn das Subnetz physisch isoliert ist (kein Gateway)
- Physische Konsole (IPMI/iDRAC) als Fallback vor Aktivierung prüfen

⚠️ **Achtung:**
- `policy_out: DROP` **nicht** ohne explizite Ausgehend-Regeln – sperrt NTP, DNS, apt-Updates
- Nach Aktivierung Corosync prüfen: `corosync-cfgtool -s`
- Security Groups ändern → `pve-firewall reload` nötig

❌ **Vermeiden:**
- `-comment "text"` in Regeln — verursacht Compile-Fehler
- `-proto` statt `-p` — funktioniert, aber weicht von UI-Standard ab
- IPsets ohne `+dc/` Prefix in Security Groups — falsche Scope-Auflösung

---

## Häufige Fehler & Lösungen

| Problem | Mögliche Ursache | Lösung |
|---------|-----------------|--------|
| `unable to parse rule parameters: -comment` | Falscher Kommentar-Syntax | `-comment "text"` → `# text` am Zeilenende |
| Web UI nicht erreichbar | Eigene IP fehlt in `mgmt-hosts` | Physische Konsole → IP eintragen → `pve-firewall reload` |
| Corosync Ring 0 unterbrochen | Port 5405 UDP blockiert | IPset `corosync-ring0` und Regel prüfen |
| Corosync Ring 1 unterbrochen | Port 5406 UDP blockiert | IPset `cluster-mgmt` und Regel prüfen |
| Live-Migration schlägt fehl | Port 60000–60050 blockiert | `cluster-mgmt` → 60000:60050 prüfen |
| Ceph OSD fällt auf `down` | Ceph-Ports blockiert | `ceph-net` → 3300/6789/6800:7300 prüfen |
| Security Group greift nicht | `pve-firewall reload` fehlt | `pve-firewall compile && pve-firewall reload` |
| SSH nach Aktivierung gesperrt | Eigene IP nicht in `mgmt-hosts` | IPMI/iDRAC nutzen → IP nachtragen |

### Debugging-Befehle

```bash
# Compile-Fehler anzeigen
pve-firewall compile

# Firewall-Log
journalctl -u pve-firewall --since "10 minutes ago" | tail -50

# iptables INPUT-Chain
iptables -L INPUT -n -v --line-numbers

# IPsets prüfen
ipset list cluster-mgmt
ipset list corosync-ring0
ipset list ceph-net

# Connectivity-Tests
nc -vzu 10.10.20.12 5405 && echo "Ring 0: OK" || echo "Ring 0: BLOCKED"
nc -vzu 172.30.7.32 5406 && echo "Ring 1: OK" || echo "Ring 1: BLOCKED"
nc -vz  10.20.20.12 3300 && echo "Ceph MON: OK" || echo "Ceph MON: BLOCKED"

ceph status
ceph osd stat
```

---

## Weiterführende Links

- [Proxmox VE Firewall – Offizielle Dokumentation](https://pve.proxmox.com/wiki/Firewall)
- [Ceph – Netzwerk-Konfiguration & Ports](https://docs.ceph.com/en/latest/rados/configuration/network-config-ref/)
- [Kapitel 02 – Netzwerk & Clustering](02-netzwerk-clustering.md)
- [Kapitel 05 – Sicherheit & Hardening](05-sicherheit-hardening.md)
