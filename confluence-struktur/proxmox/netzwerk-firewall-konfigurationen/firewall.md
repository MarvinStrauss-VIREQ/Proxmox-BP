<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: Proxmox - Netzwerk- & Firewall-Konfigurationen -->
<!-- Title: Proxmox - Firewall -->
<!-- Label: pve-sicherheit, pve-netzwerk, best-practice, referenz -->

# Firewall

> **PVE 9.x (Debian 13 Trixie)** · Letzte Aktualisierung: 2026-06
> Quelle: adaptiert aus Kapitel 07 – Firewall & IPsets. **Vor Sync gegen bestehenden Confluence-Stand prüfen.**
> Verwandte Seiten: [Port-Referenz](port-referenz.md) · [Ceph Cluster - Netzwerk](ceph-cluster-netzwerk.md) · [2 Node Cluster + QDevice - Netzwerk](2-node-cluster-qdevice-netzwerk.md)

---

## Wann anwenden?

- **Jede Neuinstallation** – Firewall aktivieren, bevor der Node produktiv geht
- **3-Node-Cluster mit Ceph** – eigenes Regelset für Monitor-, OSD- und MGR-Ports
- **2-Node-Cluster** – vereinfachtes Regelset ohne Ceph-Regeln
- **Netzwerksegmentierung** – Management-, Storage- und VM-Traffic auf Hypervisor-Ebene trennen

> **Nicht anwenden**, wenn akuter Troubleshooting-Kontext mit laufenden Ausfällen vorliegt – erst Problem lösen, dann Firewall aktivieren.

---

## Überblick

Die `pve-firewall` ist Proxmox-nativ und operiert auf drei unabhängigen Ebenen:

| Ebene | Konfigurationsdatei | Geltungsbereich |
|---|---|---|
| Datacenter | `/etc/pve/firewall/cluster.fw` | Cluster-weit (pmxcfs-repliziert) |
| Host | `/etc/pve/nodes/<node>/host.fw` | Node-spezifische Ergänzungen |
| VM/CT | `/etc/pve/firewall/<vmid>.fw` | Einzelne VMs oder Container |

### Hyper-V-Vergleich

| Konzept | Hyper-V | Proxmox VE |
|---|---|---|
| Host-Firewall | Windows Firewall (GPO) | `pve-firewall` (clusternativ) |
| IP-Gruppen | Adressbereiche im Scope | **IPsets** – cluster-weit per pmxcfs |
| Regelgruppen für VMs | kein natives Äquivalent | **Security Groups** |
| Default-Policy | Allow (manuell härten) | `DROP` ist VIREQ-Standard |

---

## Firewall-Regel-Syntax – kritische Stolperfallen

| Parameter | Falsch ❌ | Richtig ✅ | Grund |
|---|---|---|---|
| Kommentare | `-comment "SSH: Admin"` | `# SSH: Admin` am Zeilenende | `-comment` ist kein gültiger Parameter |
| Protokoll | `-proto tcp` | `-p tcp` | Kanonische Form (UI-Output) |
| Datacenter-IPsets | `+mgmt-hosts` | `+dc/mgmt-hosts` | Expliziter Scope – Pflicht in Security Groups |

```ini
IN ACCEPT -source +dc/mgmt-hosts -p tcp -dport 22 -log nolog # SSH: Admin-Zugriff
```

---

## Voraussetzungen

- Cluster aufgebaut, Corosync aktiv
- IP-Adressen aller Nodes auf allen relevanten Netzwerken bekannt
- IPMI/iDRAC oder physische Konsolenverbindung als Fallback vorab geprüft

> ⚠️ **Vor der Aktivierung:** eigene Admin-IP in `mgmt-hosts` eintragen – sonst Selbst-Lock-out.

---

## Das Firewall-Toolkit: IPsets + Security Groups

| Baustein | Gruppiert | Syntax | Einsatz |
|---|---|---|---|
| IPset | IP-Adressen | `-source +dc/cluster-mgmt` | Quell-/Ziel-Filter |
| Security Group | Firewall-Regeln | `GROUP sg-name` | Regelvorlagen für VM-Typen |

---

## Datacenter-Firewall – vollständige cluster.fw (3-Node-Ceph-Referenz)

```ini
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT
log_ratelimit: enable=1,burst=10,rate=5/second

[IPSET cluster-mgmt] # Management + Corosync Ring 1 (vmbr0 via nic0)
172.30.7.31 # pve1
172.30.7.32 # pve2
172.30.7.33 # pve3

[IPSET corosync-ring0] # Corosync Ring 0 – dedizierte NIC (ens19)
10.10.20.11 # pve1
10.10.20.12 # pve2
10.10.20.13 # pve3

[IPSET ceph-net] # bond1 – Ceph-Netz, physisch isoliert (kein Gateway)
10.20.20.0/24

[IPSET mgmt-hosts] # Admin-Workstations / Jump-Hosts – vor Aktivierung befüllen!
# 10.0.1.50

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

[RULES]
IN ACCEPT -source +dc/mgmt-hosts      -p tcp -dport 22    -log nolog # SSH: Admin-Zugriff
IN ACCEPT -source +dc/cluster-mgmt    -p tcp -dport 22    -log nolog # SSH: Inter-Node
IN ACCEPT -source +dc/mgmt-hosts      -p tcp -dport 8006  -log nolog # PVE Web UI
IN ACCEPT -source +dc/corosync-ring0  -p udp -dport 5405  -log nolog # Corosync Ring 0
IN ACCEPT -source +dc/cluster-mgmt    -p udp -dport 5406  -log nolog # Corosync Ring 1
IN ACCEPT -source +dc/cluster-mgmt    -p tcp -dport 60000:60050 -log nolog # Live-Migration
IN ACCEPT -source +dc/mgmt-hosts      -p tcp -dport 5900:5999 -log nolog # VNC/noVNC
IN ACCEPT -source +dc/mgmt-hosts      -p tcp -dport 3128  -log nolog # SPICE-Proxy
IN ACCEPT -source +dc/ceph-net        -p tcp -dport 3300  -log nolog # Ceph Monitor v2
IN ACCEPT -source +dc/ceph-net        -p tcp -dport 6789  -log nolog # Ceph Monitor v1
IN ACCEPT -source +dc/ceph-net        -p tcp -dport 6800:7300 -log nolog # Ceph OSD/MDS/MGR
IN ACCEPT -source +dc/cluster-mgmt    -p icmp -log nolog
IN ACCEPT -source +dc/corosync-ring0  -p icmp -log nolog
IN ACCEPT -source +dc/ceph-net        -p icmp -log nolog
IN ACCEPT -source +dc/mgmt-hosts      -p icmp -log nolog
```

### Einspielen

```bash
nano /etc/pve/firewall/cluster.fw
pve-firewall compile   # Dry-Run – kein Output = kein Fehler
pve-firewall reload
pve-firewall status
```

---

## Security Groups – VM-Firewall mit Vorlagen

Einmal in `cluster.fw` definieren, per `GROUP sg-name` in beliebig vielen VM-Firewalls referenzieren. Änderung wirkt nach `pve-firewall reload` sofort überall.

> Security Groups greifen ausschließlich auf VM/CT-Ebene – sie schützen den Hypervisor selbst nicht. Der `[RULES]`-Abschnitt in `cluster.fw` schützt den Hypervisor.

```ini
# /etc/pve/firewall/100.fw – Beispiel Windows-Webserver
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT

[RULES]
GROUP sg-base      # SSH + ICMP von Admin
GROUP sg-windows   # RDP + WinRM von Admin
GROUP sg-web       # HTTP/HTTPS öffentlich
```

| Security Group | Regeln | Typische VMs |
|---|---|---|
| `sg-base` | SSH + ICMP (von `mgmt-hosts`) | Alle VMs – immer erste Gruppe |
| `sg-windows` | RDP 3389, WinRM 5985/5986 | Windows Server |
| `sg-web` | HTTP 80, HTTPS 443 (öffentlich) | Webserver, Reverse Proxy |

---

## Datacenter-Firewall – 2-Node-Cluster (vereinfacht)

```ini
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT
log_ratelimit: enable=1,burst=10,rate=5/second

[IPSET cluster-mgmt]
<IP-Node1> # pve01
<IP-Node2> # pve02

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
IN ACCEPT -source +dc/cluster-mgmt  -p icmp -log nolog
IN ACCEPT -source +dc/mgmt-hosts    -p icmp -log nolog
```

---

## Firewall aktivieren – Schritt für Schritt

> ⚠️ Reihenfolge: IPsets befüllen → Dry-Run → Aktivieren → Verifizieren.

```bash
nano /etc/pve/firewall/cluster.fw   # 1. Admin-IP in mgmt-hosts eintragen
pve-firewall compile                # 2. Syntaxprüfung
pvesh set /cluster/firewall/options --enable 1   # 3. Aktivieren
pve-firewall status                 # 4. Verifizieren
iptables -L INPUT -n -v --line-numbers
ipset list cluster-mgmt
corosync-cfgtool -s                 # Beide Ringe: "connected"
ceph status                         # Kein OSD darf fallen (falls Ceph)
```

---

## Best Practices

✅ Kommentare immer als `# text` am Zeilenende – niemals `-comment "text"`
✅ IPsets immer mit `+dc/` Prefix referenzieren
✅ `sg-base` immer als erste Security Group in jeder VM-Firewall
✅ Physische Konsole (IPMI/iDRAC) als Fallback vor Aktivierung prüfen

⚠️ `policy_out: DROP` nicht ohne explizite Ausgehend-Regeln – sperrt NTP, DNS, apt-Updates
⚠️ Nach Aktivierung Corosync prüfen: `corosync-cfgtool -s`

❌ `-comment "text"` in Regeln – verursacht Compile-Fehler
❌ IPsets ohne `+dc/` Prefix in Security Groups

---

## Häufige Fehler & Lösungen

| Problem | Ursache | Lösung |
|---|---|---|
| `unable to parse rule parameters: -comment` | Falscher Kommentar-Syntax | `-comment "text"` → `# text` am Zeilenende |
| Web UI nicht erreichbar | Eigene IP fehlt in `mgmt-hosts` | Physische Konsole → IP eintragen → `pve-firewall reload` |
| Corosync unterbrochen | Port 5405/5406 UDP blockiert | IPset und Regel prüfen |
| SSH nach Aktivierung gesperrt | Eigene IP nicht in `mgmt-hosts` | IPMI/iDRAC nutzen → IP nachtragen |

---

## Weiterführende Links

| Ressource | URL |
|---|---|
| Proxmox VE Firewall – Offizielle Dokumentation | https://pve.proxmox.com/wiki/Firewall |
