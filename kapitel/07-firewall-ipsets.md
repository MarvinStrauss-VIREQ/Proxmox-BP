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

Die Datacenter-Firewall ist der zentrale Einstiegspunkt: alle IPsets, globale Policies und Standardregeln werden dort definiert und automatisch auf alle Cluster-Nodes synchronisiert.

### Hyper-V Vergleich

| Konzept | Hyper-V | Proxmox VE |
|---------|---------|------------|
| Host-Firewall | Windows Firewall (GPO oder lokale Regeln) | `pve-firewall` (clusternativ) |
| Zentrales Management | GPO / Windows Admin Center | Datacenter-Level (`cluster.fw`) |
| IP-Gruppen | Adressbereiche im Firewall-Scope | **IPsets** – einmal definiert, cluster-weit per pmxcfs |
| VM-Firewall | Hyper-V Port-ACLs (vSwitch-Ebene) | VM-Level `.fw`-Dateien unter `/etc/pve/firewall/` |
| Regelverteilung | Active Directory / GPO-Replikation | pmxcfs – automatisch, ohne AD-Abhängigkeit |
| Default-Policy | Allow (muss manuell gehärtet werden) | Konfigurierbar – `DROP` ist VIREQ-Standard |

---

## Voraussetzungen

- Proxmox VE 9.x Cluster aufgebaut und Corosync aktiv
- SSH-Zugriff auf alle Nodes (oder physische Konsole als Fallback)
- IP-Adressen aller Nodes auf **allen drei** Netzwerken bekannt
- IPMI / iDRAC / physische Konsolenverbindung als Fallback vorab prüfen

> ⚠️ **Vor der Aktivierung:** Eigene Admin-IP in `mgmt-hosts` eintragen – sonst Selbst-Lock-out.

---

## NIC-Layout (6-NIC / 3-Node-Cluster mit Ceph)

Die folgende Netzwerkstruktur gilt für jeden Node im Cluster. Gezeigt am Beispiel `pve1`:

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
└── ens23 (enp6s23)  ┘       └─ Ceph/Storage-Netz (eigenes Subnetz)
```

### Netzwerk-Übersicht pro Node

| Interface | Typ | Bond-Modus | IP (pve1 / pve2 / pve3) | Subnetz | Funktion |
|-----------|-----|-----------|--------------------------|---------|---------|
| `nic0` → `vmbr0` | Bridge | – | 172.30.7.31 / .32 / .33 | /16 | Web UI, Admin-SSH, Corosync Ring 1 |
| `ens19` | Dedizierte NIC | – | 10.10.20.11 / .12 / .13 | /24 | Corosync Ring 0 (dediziert) |
| `bond0` → `vmbr1` | Bond → Bridge | active-backup | – | – | VM-Traffic, VLAN-aware |
| `bond1` | Bond | balance-rr | 10.20.20.11 / .12 / .13 | /24 | Ceph/Storage (eigenes Subnetz) |

Drei vollständig getrennte Subnetze für drei unterschiedliche Traffic-Typen – das ist korrekt aufgebaut.

> **Hinweis bond1 `balance-rr`:** Round-Robin über beide NICs erhöht den Ceph-Durchsatz, kann aber bei TCP zu Paketumsortierung führen. Für Produktiv-Umgebungen `802.3ad` (LACP, switch-seitig) oder VIREQ-Standard `active-backup` bevorzugen.

---

## IPsets für den 3-Node-Cluster

Die drei Subnetze ergeben **vier IPsets** – je eines pro Netzwerk plus Admin-Zugriff:

| IPset-Name | Subnetz | Interface | Verwendung in Firewall-Regeln |
|------------|---------|-----------|-------------------------------|
| `cluster-mgmt` | 172.30.7.x/16 | vmbr0 / nic0 | Web UI, Admin-SSH, Corosync Ring 1, Live-Migration |
| `corosync-ring0` | 10.10.20.x/24 | ens19 | Corosync Ring 0 (nur Port 5405 UDP) |
| `ceph-net` | 10.20.20.x/24 | bond1 | Ceph Monitor, OSD, MDS, MGR |
| `mgmt-hosts` | variabel | – | Admin-Workstations, Jump-Hosts |

> **Warum kein gemeinsames `cluster-nodes`-IPset?** Corosync Ring 0 (ens19, 10.10.20.x) und Ceph-Storage (bond1, 10.20.20.x) sind auf vollständig verschiedenen Subnetzen. Würde man sie zusammenfassen, müsste man Ceph-Ports (6800–7300) für das Corosync-Subnetz öffnen und umgekehrt – unnötige Angriffsfläche.

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

[IPSET ceph-net] # Ceph/Storage-Netzwerk (bond1, eigenes Subnetz)
10.20.20.11 # pve1
10.20.20.12 # pve2
10.20.20.13 # pve3

[IPSET mgmt-hosts] # Admin-Workstations / Jump-Hosts
# 10.0.1.50   # Admin-PC (vor Aktivierung ersetzen!)
# 10.0.1.0/24 # Admin-Subnetz (optional)

########################################
# Firewall-Regeln
########################################

[RULES]

# ── SSH ──────────────────────────────────────────────────────────────────
IN ACCEPT -source +mgmt-hosts      -dport 22 -proto tcp -log nolog -comment "SSH: Admin-Zugriff"
IN ACCEPT -source +cluster-mgmt    -dport 22 -proto tcp -log nolog -comment "SSH: Inter-Node (Management-Netz)"

# ── Proxmox Web UI ───────────────────────────────────────────────────────
IN ACCEPT -source +mgmt-hosts      -dport 8006 -proto tcp -log nolog -comment "PVE Web UI (HTTPS)"

# ── Corosync Ring 0 (10.10.20.x / ens19) ────────────────────────────────
IN ACCEPT -source +corosync-ring0  -dport 5405 -proto udp -log nolog -comment "Corosync Ring 0 (ens19)"

# ── Corosync Ring 1 (172.30.7.x / vmbr0) ────────────────────────────────
IN ACCEPT -source +cluster-mgmt    -dport 5406 -proto udp -log nolog -comment "Corosync Ring 1 (vmbr0)"

# ── Live-Migration (VM/CT) ───────────────────────────────────────────────
IN ACCEPT -source +cluster-mgmt    -dport 60000:60050 -proto tcp -log nolog -comment "VM/CT Live-Migration"

# ── VNC / noVNC-Konsole ──────────────────────────────────────────────────
IN ACCEPT -source +mgmt-hosts      -dport 5900:5999 -proto tcp -log nolog -comment "VNC/noVNC-Konsole"

# ── SPICE-Proxy ──────────────────────────────────────────────────────────
IN ACCEPT -source +mgmt-hosts      -dport 3128 -proto tcp -log nolog -comment "SPICE-Proxy"

# ── Ceph Monitor (10.20.20.x / bond1) ───────────────────────────────────
IN ACCEPT -source +ceph-net        -dport 3300 -proto tcp -log nolog -comment "Ceph Monitor v2 (msgr2)"
IN ACCEPT -source +ceph-net        -dport 6789 -proto tcp -log nolog -comment "Ceph Monitor v1 (Legacy)"

# ── Ceph OSD / MDS / MGR (10.20.20.x / bond1) ──────────────────────────
IN ACCEPT -source +ceph-net        -dport 6800:7300 -proto tcp -log nolog -comment "Ceph OSD/MDS/MGR"

# ── ICMP – Monitoring & Diagnose ─────────────────────────────────────────
IN ACCEPT -source +cluster-mgmt    -proto icmp -log nolog -comment "ICMP: Management-Netz"
IN ACCEPT -source +corosync-ring0  -proto icmp -log nolog -comment "ICMP: Corosync Ring 0"
IN ACCEPT -source +ceph-net        -proto icmp -log nolog -comment "ICMP: Ceph-Netz"
IN ACCEPT -source +mgmt-hosts      -proto icmp -log nolog -comment "ICMP: Admin"
```

### Konfiguration einspielen

```bash
# Direkt bearbeiten (pmxcfs repliziert automatisch an alle Nodes)
nano /etc/pve/firewall/cluster.fw

# Syntaxprüfung vor dem Aktivieren (kein Reload, nur Prüfung)
pve-firewall compile

# Regeln neu laden
pve-firewall reload

# Status anzeigen
pve-firewall status
```

---

## Datacenter-Firewall – 2-Node-Cluster (vereinfacht)

Für 2-Node-Cluster ohne Ceph (PBS als QDevice, kein dediziertes Storage-Netz):

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
IN ACCEPT -source +mgmt-hosts    -dport 22 -proto tcp -log nolog -comment "SSH: Admin-Zugriff"
IN ACCEPT -source +cluster-mgmt  -dport 22 -proto tcp -log nolog -comment "SSH: Inter-Node"
IN ACCEPT -source +mgmt-hosts    -dport 8006 -proto tcp -log nolog -comment "PVE Web UI"
IN ACCEPT -source +cluster-mgmt  -dport 5405 -proto udp -log nolog -comment "Corosync Ring 0"
IN ACCEPT -source +cluster-mgmt  -dport 60000:60050 -proto tcp -log nolog -comment "VM/CT Live-Migration"
IN ACCEPT -source +mgmt-hosts    -dport 5900:5999 -proto tcp -log nolog -comment "VNC/noVNC-Konsole"
IN ACCEPT -source +mgmt-hosts    -dport 3128 -proto tcp -log nolog -comment "SPICE-Proxy"
IN ACCEPT -source +cluster-mgmt  -proto icmp -log nolog -comment "ICMP: Inter-Node"
IN ACCEPT -source +mgmt-hosts    -proto icmp -log nolog -comment "ICMP: Admin"
```

---

## Host-Firewall konfigurieren

Die `host.fw` ergänzt die Datacenter-Regeln pro Node. In einem homogenen Cluster meist leer.

```ini
[OPTIONS]
enable: 1
# Datacenter-Regeln (cluster.fw) gelten bereits.
# Hier nur node-spezifische Ergänzungen eintragen.

[RULES]
# Prometheus Node Exporter (optional – nur wenn externes Monitoring aktiv)
# IN ACCEPT -source +mgmt-hosts -dport 9100 -proto tcp -log nolog -comment "Prometheus Node Exporter"

# Ceph Dashboard (direkt über Port 8443, optional)
# Normalerweise via PVE Web UI (Port 8006) ausreichend
# IN ACCEPT -source +mgmt-hosts -dport 8443 -proto tcp -log nolog -comment "Ceph Dashboard direkt"

# NinjaOne Agent: keine eingehenden Ports – Agent baut ausgehende Verbindung auf
```

```bash
# host.fw für jeden Node individuell
nano /etc/pve/nodes/pve1/host.fw
nano /etc/pve/nodes/pve2/host.fw
nano /etc/pve/nodes/pve3/host.fw
```

---

## Firewall aktivieren – Schritt für Schritt

> ⚠️ **Reihenfolge einhalten:** IPsets befüllen → Dry-run → Aktivieren → Verifizieren.

### Schritt 1: Admin-IP eintragen

```bash
# Eigene IP ermitteln (Zugriff erfolgt über Management-Netz 172.30.7.x)
ip route get 172.30.7.31 | awk '{print $7; exit}'

# cluster.fw bearbeiten – eigene IP in [IPSET mgmt-hosts] eintragen
nano /etc/pve/firewall/cluster.fw
```

### Schritt 2: Syntaxprüfung

```bash
pve-firewall compile
# Keine Fehlermeldung = Syntax korrekt
```

### Schritt 3: Aktivieren

```bash
# Per CLI
pvesh set /cluster/firewall/options --enable 1

# Oder: Web UI → Datacenter → Firewall → Options → Firewall: Yes
```

### Schritt 4: Verifizieren

```bash
# Firewall-Status
pve-firewall status

# iptables – aktive INPUT-Regeln
iptables -L INPUT -n -v --line-numbers

# IPsets prüfen
ipset list cluster-mgmt
ipset list corosync-ring0
ipset list ceph-net
ipset list mgmt-hosts

# Corosync – beide Ringe müssen "connected" sein
corosync-cfgtool -s

# Ceph-Health nach Aktivierung (kein OSD darf abfallen)
ceph status
ceph osd stat
```

### Schritt 5: Cluster-weite Statusprüfung

```bash
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
| 3300 | TCP | Ceph Monitor v2 (msgr2) | `ceph-net` | bond1 | ✅ (Ceph) |
| 6789 | TCP | Ceph Monitor v1 | `ceph-net` | bond1 | ✅ (Ceph) |
| 6800–7300 | TCP | Ceph OSD / MDS / MGR | `ceph-net` | bond1 | ✅ (Ceph) |
| — | ICMP | Ping / Monitoring | alle IPsets | alle | ✅ |
| 9100 | TCP | Prometheus Node Exporter | `mgmt-hosts` | vmbr0 | Optional |
| 8443 | TCP | Ceph Dashboard (direkt) | `mgmt-hosts` | vmbr0 | Optional |
| 4789 | UDP | VXLAN (SDN) | `cluster-mgmt` | vmbr0 | Optional |

---

## Ceph-Referenz (Cluster-Status)

Für den Abgleich mit dem Ist-Zustand: typische Ceph-Konfiguration eines 3-Node-Clusters.

| Parameter | Wert (Referenz-Cluster) | Hinweis |
|-----------|------------------------|---------|
| Ceph Version | 19.2.3 (Squid) | Aktuell mit PVE 9.x |
| OSDs gesamt | 12 | 4 OSDs pro Node |
| Monitors | 3 (pve1, pve2, pve3) | 1 Monitor pro Node |
| Manager | 2 aktiv | Automatisches Failover |
| OSD-Port-Range | 6800–7300/TCP | Automatisch zugewiesen |
| Monitor-Port | 3300/TCP (v2), 6789/TCP (v1) | v2 (msgr2) bevorzugt |

```bash
# Ceph-Version prüfen
ceph version

# OSD-Belegung pro Node anzeigen
ceph osd tree

# Monitor-Status
ceph mon stat

# Manager-Status
ceph mgr stat
```

---

## Best Practices

✅ **Empfohlen:**
- `policy_in: DROP` als Datacenter-Standard – Whitelist-Ansatz
- Drei getrennte Cluster-IPsets (`cluster-mgmt`, `corosync-ring0`, `ceph-net`) – je ein IPset pro Subnetz
- Ceph-Ports (6800–7300) ausschließlich für `ceph-net` öffnen, nicht für Corosync-IPs
- `log_ratelimit` aktivieren – verhindert Log-Flooding bei Port-Scans
- Nach Cluster-Erweiterung alle IPsets sofort aktualisieren
- Physische Konsole (IPMI/iDRAC) als Fallback vor Aktivierung prüfen

⚠️ **Achtung:**
- `policy_out: DROP` **nicht** ohne explizite Ausgehend-Regeln – sperrt NTP, DNS, apt-Updates
- Nach Aktivierung immer Corosync-Status prüfen: `corosync-cfgtool -s` (beide Ringe connected?)
- `mgmt-hosts` bei Änderung der Admin-IP sofort aktualisieren
- bond1 `balance-rr`: in Produktiv-Umgebung auf `802.3ad` (LACP) oder VIREQ-Standard `active-backup` wechseln

❌ **Vermeiden:**
- `policy_in: ACCEPT` im Produktivbetrieb
- Corosync Ring 0 und Ceph in einem gemeinsamen IPset zusammenfassen (unterschiedliche Subnetze, unterschiedliche erlaubte Ports)
- VNC-Ports (5900–5999) ohne Quell-IP-Einschränkung öffnen

---

## Häufige Fehler & Lösungen

| Problem | Mögliche Ursache | Lösung |
|---------|-----------------|--------|
| Web UI nicht erreichbar | Eigene IP fehlt in `mgmt-hosts` | Physische Konsole → IP eintragen → `pve-firewall reload` |
| Corosync Ring 0 unterbrochen | Port 5405 UDP von `corosync-ring0` blockiert | IPset `corosync-ring0` und Regel prüfen |
| Corosync Ring 1 unterbrochen | Port 5406 UDP von `cluster-mgmt` blockiert | IPset `cluster-mgmt` und Regel prüfen |
| Live-Migration schlägt fehl | Port 60000–60050 nicht freigegeben | `cluster-mgmt` → 60000:60050; auf Ziel-Node `iptables -L -n` |
| Ceph OSD fällt auf `down` | Ceph-Ports blockiert | `ceph-net` → 3300/6789/6800:7300 prüfen |
| `pve-firewall compile` schlägt fehl | Syntaxfehler | Fehlermeldung auswerten – häufig: `-comment` ohne Anführungszeichen |
| SSH nach Aktivierung gesperrt | Eigene IP nicht in `mgmt-hosts` | IPMI/iDRAC nutzen → IP nachtragen |

### Debugging-Befehle

```bash
# Firewall-Log (letzte 10 Minuten)
journalctl -u pve-firewall --since "10 minutes ago" | tail -50

# iptables – INPUT-Chain
iptables -L INPUT -n -v --line-numbers

# Alle IPsets prüfen
ipset list cluster-mgmt
ipset list corosync-ring0
ipset list ceph-net

# Corosync Ring 0 (ens19 / 10.10.20.x) testen
nc -vzu 10.10.20.12 5405 && echo "Ring 0: OK" || echo "Ring 0: BLOCKED"

# Corosync Ring 1 (vmbr0 / 172.30.7.x) testen
nc -vzu 172.30.7.32 5406 && echo "Ring 1: OK" || echo "Ring 1: BLOCKED"

# Ceph-Erreichbarkeit auf Storage-Netz testen
nc -vz 10.20.20.12 3300 && echo "Ceph MON: OK" || echo "Ceph MON: BLOCKED"

# Ceph-Status nach Firewall-Änderungen
ceph status
ceph osd stat
```

---

## Weiterführende Links

- [Proxmox VE Firewall – Offizielle Dokumentation](https://pve.proxmox.com/wiki/Firewall)
- [Ceph – Netzwerk-Konfiguration & Ports](https://docs.ceph.com/en/latest/rados/configuration/network-config-ref/)
- [Kapitel 02 – Netzwerk & Clustering](02-netzwerk-clustering.md)
- [Kapitel 05 – Sicherheit & Hardening](05-sicherheit-hardening.md)
