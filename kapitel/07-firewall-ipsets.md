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
- **2-Node-Cluster** – Vereinfachtes Regelset (kein Ceph-Netz erforderlich, Abschnitt unten)
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

Die Datacenter-Firewall ist der zentrale Einstiegspunkt: alle IPsets, globale Policies und Standardregeln werden dort definiert und automatisch auf alle Cluster-Nodes synchronisiert — kein manueller Abgleich zwischen Nodes nötig.

### Hyper-V Vergleich

| Konzept | Hyper-V | Proxmox VE |
|---------|---------|------------|
| Host-Firewall | Windows Firewall (GPO oder lokale Regeln) | `pve-firewall` (clusternativ) |
| Zentrales Management | GPO / Windows Admin Center | Datacenter-Level (`cluster.fw`) |
| IP-Gruppen | Adressbereiche im Firewall-Scope | **IPsets** – einmal definiert, cluster-weit verwendbar |
| VM-Firewall | Hyper-V Port-ACLs (vSwitch-Ebene) | VM-Level `.fw`-Dateien unter `/etc/pve/firewall/` |
| Regelverteilung | Active Directory / GPO-Replikation | pmxcfs (automatisch, ohne AD-Abhängigkeit) |
| Default-Policy | Allow (muss manuell gehärtet werden) | Konfigurierbar — `DROP` ist VIREQ-Standard |

> **Kernunterschied:** IPsets in Proxmox sind cluster-weit gültig und werden automatisch repliziert. Kein manuelles Verteilen wie bei GPO-basierten Firewall-Regeln.

---

## Voraussetzungen

- Proxmox VE 9.x Cluster aufgebaut und Corosync aktiv
- SSH-Zugriff auf alle Nodes (oder physische Konsole als Fallback)
- IP-Adressen aller Nodes **und** der eigenen Admin-Workstation bekannt
- Netzwerktopologie dokumentiert (Management-Netz vs. Storage-/Ceph-Netz)

> ⚠️ **Vor der Aktivierung:** Eigene Admin-IP in das IPset `mgmt-hosts` eintragen — sonst Selbst-Lock-out.  
> ⚠️ **Fallback sicherstellen:** IPMI / iDRAC / physische Konsolenverbindung prüfen.

---

## Netzwerk-Topologie (3-Node-Cluster mit Ceph)

Aus der Proxmox-Clusteransicht (Datacenter → Cluster) ergibt sich die Netzstruktur. Beispiel-Cluster `ceph-test` mit zwei Netzwerken:

| Node | Link 0 – Management / Corosync Ring 0 | Link 1 – Ceph-Netz / Corosync Ring 1 |
|------|--------------------------------------|--------------------------------------|
| pve1 | 10.10.20.11 | 172.30.7.31 |
| pve2 | 10.10.20.12 | 172.30.7.32 |
| pve3 | 10.10.20.13 | 172.30.7.33 |

Daraus ergeben sich **drei IPsets** für die Firewall:

| IPset-Name | Netzwerk | Inhalt |
|------------|----------|--------|
| `cluster-nodes` | 10.10.20.0/24 | Management-IPs der Nodes (Link 0 / Corosync Ring 0) |
| `ceph-net` | 172.30.7.0/24 | Ceph-Storage-IPs (Link 1 / Corosync Ring 1) |
| `mgmt-hosts` | variabel | Admin-Workstations, Jump-Hosts, Monitoring-Server |

---

## Datacenter-Firewall konfigurieren

Die Datei `/etc/pve/firewall/cluster.fw` ist der zentrale Ort für alle Cluster-weiten Firewall-Regeln und IPsets. Sie wird durch pmxcfs automatisch auf alle Nodes repliziert — Änderungen auf einem Node wirken sofort auf allen.

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

[IPSET cluster-nodes] # Management-Netzwerk (Link 0 / Corosync Ring 0)
10.10.20.11 # pve1
10.10.20.12 # pve2
10.10.20.13 # pve3

[IPSET ceph-net] # Ceph/Storage-Netzwerk (Link 1 / Corosync Ring 1)
172.30.7.31 # pve1
172.30.7.32 # pve2
172.30.7.33 # pve3

[IPSET mgmt-hosts] # Admin-Workstations / Jump-Hosts
# 10.0.1.50   # Admin-PC (Beispiel – vor Aktivierung ersetzen!)
# 10.0.1.0/24 # Admin-Subnetz (Beispiel)

########################################
# Firewall-Regeln
########################################

[RULES]

# ── SSH ──────────────────────────────────────────────────────────────────
IN ACCEPT -source +mgmt-hosts    -dport 22 -proto tcp -log nolog -comment "SSH: Admin-Zugriff"
IN ACCEPT -source +cluster-nodes -dport 22 -proto tcp -log nolog -comment "SSH: Inter-Node (Cluster-Ops)"

# ── Proxmox Web UI ───────────────────────────────────────────────────────
IN ACCEPT -source +mgmt-hosts    -dport 8006 -proto tcp -log nolog -comment "PVE Web UI (HTTPS)"

# ── Corosync – Cluster-Kommunikation ────────────────────────────────────
IN ACCEPT -source +cluster-nodes -dport 5405 -proto udp -log nolog -comment "Corosync Ring 0 (Link 0)"
IN ACCEPT -source +ceph-net      -dport 5406 -proto udp -log nolog -comment "Corosync Ring 1 (Link 1)"

# ── Live-Migration (VM/CT) ───────────────────────────────────────────────
IN ACCEPT -source +cluster-nodes -dport 60000:60050 -proto tcp -log nolog -comment "VM/CT Live-Migration"

# ── VNC / noVNC-Konsole ──────────────────────────────────────────────────
IN ACCEPT -source +mgmt-hosts    -dport 5900:5999 -proto tcp -log nolog -comment "VNC/noVNC-Konsole"

# ── SPICE-Proxy ──────────────────────────────────────────────────────────
IN ACCEPT -source +mgmt-hosts    -dport 3128 -proto tcp -log nolog -comment "SPICE-Proxy"

# ── Ceph Monitor ─────────────────────────────────────────────────────────
IN ACCEPT -source +ceph-net      -dport 3300 -proto tcp -log nolog -comment "Ceph Monitor v2 (msgr2)"
IN ACCEPT -source +ceph-net      -dport 6789 -proto tcp -log nolog -comment "Ceph Monitor v1 (Legacy)"

# ── Ceph OSD / MDS / MGR ────────────────────────────────────────────────
IN ACCEPT -source +ceph-net      -dport 6800:7300 -proto tcp -log nolog -comment "Ceph OSD/MDS/MGR"

# ── ICMP – Monitoring & Diagnose ─────────────────────────────────────────
IN ACCEPT -source +cluster-nodes -proto icmp -log nolog -comment "ICMP: Inter-Node"
IN ACCEPT -source +ceph-net      -proto icmp -log nolog -comment "ICMP: Ceph-Netz"
IN ACCEPT -source +mgmt-hosts    -proto icmp -log nolog -comment "ICMP: Admin"
```

### Konfiguration einspielen

```bash
# Direkt bearbeiten (pmxcfs repliziert automatisch an alle Nodes)
nano /etc/pve/firewall/cluster.fw

# Syntaxprüfung vor dem Aktivieren (Dry-run – keine Änderungen)
pve-firewall compile

# Regeln neu laden (nach Änderungen)
pve-firewall reload

# Aktuellen Status anzeigen
pve-firewall status
```

---

## Datacenter-Firewall – 2-Node-Cluster (vereinfacht)

Für 2-Node-Cluster ohne dediziertes Ceph-Netz (PBS als QDevice, kein separates Storage-Netz):

```ini
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT
log_ratelimit: enable=1,burst=10,rate=5/second

[IPSET cluster-nodes]
10.10.20.11 # pve1
10.10.20.12 # pve2
# Optional: PBS-IP eintragen falls als QDevice genutzt
# 10.10.20.20 # pbs1

[IPSET mgmt-hosts]
# 10.0.1.50 # Admin-PC

[RULES]

IN ACCEPT -source +mgmt-hosts    -dport 22 -proto tcp -log nolog -comment "SSH: Admin-Zugriff"
IN ACCEPT -source +cluster-nodes -dport 22 -proto tcp -log nolog -comment "SSH: Inter-Node"
IN ACCEPT -source +mgmt-hosts    -dport 8006 -proto tcp -log nolog -comment "PVE Web UI"
IN ACCEPT -source +cluster-nodes -dport 5405 -proto udp -log nolog -comment "Corosync Ring 0"
IN ACCEPT -source +cluster-nodes -dport 60000:60050 -proto tcp -log nolog -comment "VM/CT Live-Migration"
IN ACCEPT -source +mgmt-hosts    -dport 5900:5999 -proto tcp -log nolog -comment "VNC/noVNC-Konsole"
IN ACCEPT -source +mgmt-hosts    -dport 3128 -proto tcp -log nolog -comment "SPICE-Proxy"
IN ACCEPT -source +cluster-nodes -proto icmp -log nolog -comment "ICMP: Inter-Node"
IN ACCEPT -source +mgmt-hosts    -proto icmp -log nolog -comment "ICMP: Admin"
```

---

## Host-Firewall konfigurieren

Die `host.fw` ergänzt die Datacenter-Regeln pro Node. In einem homogenen Cluster ist sie meist leer oder enthält nur wenige Erweiterungen.

### Wann host.fw verwenden?

- Node-spezifische Dienste (z.B. Prometheus Node Exporter nur auf einem Node)
- Temporäre Debug-Regeln (z.B. breiter öffnen für Troubleshooting)
- Abweichende Admin-Zugänge auf einzelnen Nodes

### Minimalkonfiguration (für alle 3 Nodes identisch)

```ini
[OPTIONS]
enable: 1
# Datacenter-Regeln (cluster.fw) gelten bereits.
# Hier nur node-spezifische Ergänzungen eintragen.

[RULES]
# Prometheus Node Exporter (optional – nur wenn externes Monitoring-System)
# IN ACCEPT -source +mgmt-hosts -dport 9100 -proto tcp -log nolog -comment "Prometheus Node Exporter"

# Ceph Dashboard (direkt, optional – normalerweise über PVE Web UI auf 8006)
# IN ACCEPT -source +mgmt-hosts -dport 8443 -proto tcp -log nolog -comment "Ceph Dashboard (direkt)"

# NinjaOne Agent: keine eingehenden Ports nötig (Agent baut ausgehende Verbindung auf)
```

```bash
# host.fw für jeden Node individuell setzen
nano /etc/pve/nodes/pve1/host.fw
nano /etc/pve/nodes/pve2/host.fw
nano /etc/pve/nodes/pve3/host.fw
```

---

## Firewall aktivieren – Schritt für Schritt

> ⚠️ **Reihenfolge einhalten:** Erst IPsets befüllen → Dry-run → Aktivieren → Verifizieren.

### Schritt 1: Admin-IP eintragen

```bash
# Eigene öffentliche oder interne IP ermitteln (auf Admin-PC ausführen)
curl -s ifconfig.me   # Falls Admin über Internet
ip route get 10.10.20.11 | awk '{print $7; exit}'  # Falls Admin im lokalen Netz

# cluster.fw bearbeiten und eigene IP in [IPSET mgmt-hosts] eintragen
nano /etc/pve/firewall/cluster.fw
```

### Schritt 2: Syntaxprüfung (Dry-run)

```bash
# Konfiguration kompilieren ohne anzuwenden – zeigt Syntaxfehler
pve-firewall compile

# Erwartete Ausgabe: Objekte ohne Fehlermeldungen
# Fehler: z.B. fehlende Anführungszeichen in -comment oder ungültige IP-Syntax
```

### Schritt 3: Firewall aktivieren

```bash
# Option A: Per CLI (empfohlen)
pvesh set /cluster/firewall/options --enable 1

# Option B: Web UI
# Datacenter → Firewall → Options → Firewall: Yes → OK
```

### Schritt 4: Verifizieren

```bash
# Firewall-Status auf dem aktuellen Node
pve-firewall status

# Aktive iptables-Regeln (INPUT-Chain)
iptables -L INPUT -n -v --line-numbers

# IPset-Inhalt prüfen
ipset list cluster-nodes
ipset list ceph-net
ipset list mgmt-hosts

# Corosync-Ringe nach Aktivierung validieren (alle müssen "connected" sein)
corosync-cfgtool -s

# Ceph-Status prüfen (kein OSD darf auf "down" fallen)
ceph status
ceph osd tree
```

### Schritt 5: Status auf allen Nodes prüfen

```bash
# Firewall-Status cluster-weit abfragen
for node in pve1 pve2 pve3; do
  echo "=== $node ==="
  ssh root@$node "pve-firewall status && corosync-cfgtool -s | grep -E 'ring|FAULTY'"
done
```

---

## Port-Referenz

| Port(s) | Protokoll | Dienst | Quelle (IPset) | Erforderlich |
|---------|-----------|--------|---------------|:---:|
| 22 | TCP | SSH | `mgmt-hosts`, `cluster-nodes` | ✅ |
| 8006 | TCP | Proxmox Web UI (HTTPS) | `mgmt-hosts` | ✅ |
| 5405 | UDP | Corosync Ring 0 | `cluster-nodes` | ✅ |
| 5406 | UDP | Corosync Ring 1 | `ceph-net` | ✅ (Ceph) |
| 60000–60050 | TCP | VM/CT Live-Migration | `cluster-nodes` | ✅ |
| 5900–5999 | TCP | VNC / noVNC-Konsole | `mgmt-hosts` | ✅ |
| 3128 | TCP | SPICE-Proxy | `mgmt-hosts` | ✅ |
| 3300 | TCP | Ceph Monitor v2 (msgr2) | `ceph-net` | ✅ (Ceph) |
| 6789 | TCP | Ceph Monitor v1 (Legacy) | `ceph-net` | ✅ (Ceph) |
| 6800–7300 | TCP | Ceph OSD / MDS / MGR | `ceph-net` | ✅ (Ceph) |
| — | ICMP | Ping / Monitoring | alle IPsets | ✅ |
| 9100 | TCP | Prometheus Node Exporter | `mgmt-hosts` | Optional |
| 8443 | TCP | Ceph Dashboard (direkt) | `mgmt-hosts` | Optional |
| 4789 | UDP | VXLAN (SDN) | `cluster-nodes` | Optional (SDN) |

---

## Best Practices

✅ **Empfohlen:**
- `policy_in: DROP` als Datacenter-Standard — Whitelist-Ansatz ist sicherer als Blacklisting
- IPsets für alle Netzwerkgruppen definieren — niemals IP-Adressen direkt in Regeln hardcoden
- `log_ratelimit` aktivieren — verhindert Log-Flooding bei Port-Scans
- Firewall-Änderungen immer mit `pve-firewall compile` prüfen bevor `reload`
- Nach Cluster-Erweiterung (neuer Node) alle relevanten IPsets aktualisieren
- Physische Konsolenverbindung (IPMI/iDRAC) vor Aktivierung sicherstellen

⚠️ **Achtung:**
- `policy_out: DROP` **nicht** ohne explizite Ausgehend-Regeln setzen — blockiert NTP, DNS und apt-Updates
- Ceph-Regeln ausschließlich auf das Ceph-Netz (`ceph-net`) begrenzen — **nicht** allgemein für alle IPs öffnen
- Nach Firewall-Aktivierung immer Corosync-Status prüfen (`corosync-cfgtool -s`)
- `mgmt-hosts` bei Änderung der Admin-IP sofort aktualisieren, bevor der Zugang verloren geht

❌ **Vermeiden:**
- `policy_in: ACCEPT` im Produktivbetrieb (alles erlauben)
- VNC-Ports (5900–5999) ohne Quell-IP-Einschränkung öffnen
- IP-Adressen direkt in Regeln ohne IPset (nicht wartbar bei Cluster-Erweiterung)
- Firewall auf einem einzelnen Node deaktivieren und nicht wieder aktivieren

---

## Häufige Fehler & Lösungen

| Problem | Mögliche Ursache | Lösung |
|---------|-----------------|--------|
| Web UI nicht mehr erreichbar | Eigene IP fehlt in `mgmt-hosts` | Physische Konsole → IP eintragen → `pve-firewall reload` |
| Corosync-Verbindung bricht ab | Port 5405/5406 UDP blockiert | `iptables -L INPUT -n` prüfen; Corosync-Regeln kontrollieren |
| Live-Migration schlägt fehl | Port 60000–60050 nicht freigegeben | Auf Ziel-Node: `iptables -L INPUT -n`; `cluster-nodes`-Regel prüfen |
| Ceph OSD fällt auf `down` | Port 6800–7300 oder 3300/6789 blockiert | `ceph health detail`; `ipset list ceph-net` überprüfen |
| `pve-firewall compile` schlägt fehl | Syntaxfehler in `cluster.fw` | Fehlermeldung auswerten — häufig: Anführungszeichen in `-comment` fehlen |
| SSH nach Aktivierung gesperrt | Eigene IP nicht in `mgmt-hosts` | IPMI/iDRAC nutzen; IP nachtragen; `pve-firewall reload` |
| ICMP (Ping) schlägt fehl | ICMP-Regeln fehlen oder falscher IPset | ICMP-Regeln für relevante IPsets prüfen |

### Debugging-Befehle

```bash
# Firewall-Log (Drops/Rejects der letzten 10 Minuten)
journalctl -u pve-firewall --since "10 minutes ago" | tail -50

# Aktive iptables-Regeln (INPUT-Chain, alle Nodes)
iptables -L INPUT -n -v --line-numbers

# IPset-Inhalt prüfen
ipset list cluster-nodes
ipset list ceph-net
ipset list mgmt-hosts

# Corosync-Ringe nach Firewall-Änderung prüfen
corosync-cfgtool -s

# Ceph-Cluster-Health nach Firewall-Aktivierung
ceph status
ceph osd tree

# Verbindung von pve2 zu pve1 auf Corosync-Port testen
ssh root@pve2 "nc -vzu 10.10.20.11 5405 && echo OK || echo BLOCKED"
ssh root@pve2 "nc -vzu 172.30.7.31 5406 && echo OK || echo BLOCKED"
```

---

## Weiterführende Links

- [Proxmox VE Firewall – Offizielle Dokumentation](https://pve.proxmox.com/wiki/Firewall)
- [Ceph – Netzwerk-Konfiguration & Ports](https://docs.ceph.com/en/latest/rados/configuration/network-config-ref/)
- [Kapitel 02 – Netzwerk & Clustering](02-netzwerk-clustering.md)
- [Kapitel 05 – Sicherheit & Hardening](05-sicherheit-hardening.md)
