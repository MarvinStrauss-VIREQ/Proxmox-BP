<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: PVE – Best Practices -->
<!-- Title: PVE – Storage: Ceph Setup & Konfiguration -->
<!-- Label: pve-storage, best-practice, referenz, pve-cluster -->

# Kapitel 09: Ceph Storage

> **PVE 9.x / Ceph 19.x Squid (Debian 13 Trixie)** · Letzte Aktualisierung: 2026-06  
> Verwandte Kapitel: [02 – Netzwerk & Clustering](02-netzwerk-clustering.md) · [10 – High Availability](10-high-availability.md)

---

## Wann anwenden?

- **3-Node-Cluster** – Ceph ist der Standard für shared Storage in Proxmox-Clustern ab 3 Nodes
- **Maximale Ausfallsicherheit** – Ceph mit `size=3` übersteht den Komplettausfall eines Nodes ohne Datenverlust
- **Live-Migration & HA** – Voraussetzung für VM-Live-Migration und HA-Manager (shared Storage erforderlich)

> **Nicht für 2-Node-Cluster** – Ceph benötigt mindestens 3 Nodes für sicheren Betrieb (Quorum der Ceph-Monitore). Für 2-Node: NFS oder PBS als Storage.

---

## Überblick

Ceph ist ein verteiltes Software-Storage-System, das Redundanz direkt im Speicher-Layer verankert — ohne dedizierte Storage-Hardware. Jede VM-Disk wird in konfigurierten Replikas auf mehrere physische Nodes verteilt.

```
VM schreibt Daten
      ↓
  Ceph RBD (Proxmox Storage-Backend)
      ↓
  Ceph Pool (vm-store, size=3, CRUSH-Regel: host-Ebene)
      ↓
  Replica 1 → OSD auf pve1    Replica 2 → OSD auf pve2    Replica 3 → OSD auf pve3
```

Fällt pve1 aus: Ceph arbeitet weiter mit den verbleibenden 2 Repliken (`min_size=2`). Daten bleiben vollständig erhalten.

### Hyper-V Vergleich

| Konzept | Hyper-V / Windows | Proxmox / Ceph |
|---------|------------------|----------------|
| Shared Storage | SAN / iSCSI / SMB 3.0 | Ceph RBD (nativ im Cluster) |
| Redundanz | RAID-Controller, externes SAN | Software-RAID über alle Nodes |
| Ausfallsicherheit | Abhängig von SAN-Hardware | 1 Node-Ausfall bei size=3 ohne Datenverlust |
| Skalierung | Neue SAN-Hardware nötig | Node/OSD hinzufügen → automatisch balanciert |
| Kosten | SAN-Lizenz + Hardware | Keine Lizenzkosten, Commodity-Hardware |
| Live-Migration | Shared SAN/SMB erforderlich | Ceph RBD nativ unterstützt |

> **Storage Spaces Direct (S2D)** ist der nächste Vergleichspunkt: S2D macht ähnliches wie Ceph (verteilter Storage über lokale Disks), ist aber ausschließlich auf Windows beschränkt und lizenzpflichtig.

---

## Ceph-Architektur (3-Node-Cluster)

| Komponente | Anzahl | Beschreibung |
|------------|--------|--------------|
| **MON** (Monitor) | 3 (1 pro Node) | Cluster-Map, Quorum, Auth — Ceph-eigener Quorum |
| **MGR** (Manager) | 2 (aktiv + standby) | Metriken, Dashboard, Module |
| **OSD** (Object Storage Daemon) | 12 (4 pro Node) | Speichert tatsächliche Daten |
| **MDS** (Metadata Server) | optional | Nur für CephFS benötigt |

**Ceph-Netzwerk in diesem Cluster:**

| Netzwerk | Subnetz | Interface | Funktion |
|----------|---------|-----------|---------|
| Public Network | 10.20.20.0/24 | bond1 | Client-I/O (Proxmox ↔ Ceph) |
| Cluster Network | 10.20.20.0/24 | bond1 | OSD-Replikation (OSD ↔ OSD) |

> **Produktiv-Empfehlung:** Public und Cluster Network auf getrennten Subnetzen für optimale Performance. Im Blueprint wird ein kombiniertes Netz (10.20.20.x) verwendet — für die meisten SMB-Deployments akzeptabel.

---

## Voraussetzungen

- 3 Nodes mit Corosync-Cluster (Kapitel 02)
- Ceph-Netz auf bond1 konfiguriert (10.20.20.x/24)
- Mindestens 1 dedizierte Disk pro OSD (keine OS-Disk!)
- Für Produktion: Enterprise-SSDs mit Power Loss Protection (PLP) oder SAS-HDDs
- `/dev/watchdog` verfügbar auf allen Nodes (für HA in Kapitel 10)

```bash
# Ceph-Package-Status prüfen (PVE 9.x: ceph-squid vorinstalliert)
pveceph status

# Verfügbare Disks pro Node prüfen
lsblk -d -o NAME,SIZE,ROTA,TYPE | grep disk
# ROTA=1 = HDD, ROTA=0 = SSD/NVMe
```

---

## Schritt 1: Ceph initialisieren

Einmalig auf dem **ersten Node** (`pve1`) ausführen:

```bash
# Ceph mit Netzwerkkonfiguration initialisieren
pveceph init --network 10.20.20.0/24

# Erzeugte Konfiguration prüfen
cat /etc/ceph/ceph.conf
```

Erwartete `ceph.conf` nach Init:

```ini
[global]
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
public network = 10.20.20.0/24
cluster network = 10.20.20.0/24
```

---

## Schritt 2: Monitore einrichten

Je **einen Monitor pro Node** — der Monitor-Quorum ist der Kern des Ceph-Clusters:

```bash
# Monitor auf pve1 (bereits durch Init erstellt)
pveceph mon create

# Monitor auf pve2 (SSH nach pve2 oder via Web UI)
ssh root@pve2 "pveceph mon create"

# Monitor auf pve3
ssh root@pve3 "pveceph mon create"

# Monitor-Status prüfen (alle 3 müssen "in quorum" sein)
ceph mon stat
# Erwartete Ausgabe: "3 mons at ..., election epoch ..., quorum 0,1,2 pve1,pve2,pve3"
```

**Web UI:** Datacenter → Ceph → Monitor → Add (auf jedem Node wiederholen)

---

## Schritt 3: Manager einrichten

```bash
# Manager auf pve1 und pve2 (aktiv + standby)
pveceph mgr create
ssh root@pve2 "pveceph mgr create"

# Status: einer aktiv, einer standby
ceph mgr stat
```

---

## Schritt 4: OSDs erstellen

Jede dedizierte Disk wird ein OSD. Im Referenz-Cluster: **4 Disks pro Node = 12 OSDs gesamt**.

```bash
# OSDs auf pve1 erstellen (alle verfügbaren Disks)
pveceph osd create /dev/sdb --crush-device-class hdd
pveceph osd create /dev/sdc --crush-device-class hdd
pveceph osd create /dev/sdd --crush-device-class hdd
pveceph osd create /dev/sde --crush-device-class hdd

# Auf pve2 wiederholen
ssh root@pve2 "pveceph osd create /dev/sdb --crush-device-class hdd"
ssh root@pve2 "pveceph osd create /dev/sdc --crush-device-class hdd"
ssh root@pve2 "pveceph osd create /dev/sdd --crush-device-class hdd"
ssh root@pve2 "pveceph osd create /dev/sde --crush-device-class hdd"

# Auf pve3 wiederholen
ssh root@pve3 "pveceph osd create /dev/sdb --crush-device-class hdd"
ssh root@pve3 "pveceph osd create /dev/sdc --crush-device-class hdd"
ssh root@pve3 "pveceph osd create /dev/sdd --crush-device-class hdd"
ssh root@pve3 "pveceph osd create /dev/sde --crush-device-class hdd"

# OSD-Baum prüfen: alle 12 OSDs auf je 3 Hosts verteilt
ceph osd tree
```

Erwartete Ausgabe `ceph osd tree`:
```
ID  CLASS  WEIGHT   TYPE NAME
-1         gesamt   root default
-3         subtotal   host pve1
 0   hdd   weight     osd.0  (up, in)
 1   hdd   weight     osd.1  (up, in)
 2   hdd   weight     osd.2  (up, in)
 3   hdd   weight     osd.3  (up, in)
-5         subtotal   host pve2
 4   hdd   weight     osd.4  (up, in)
...
```

> **Wichtig:** `--crush-device-class hdd` klassifiziert die OSD korrekt. Bei gemischten SSD/HDD-Setups ermöglicht das class-basierte CRUSH-Regeln (z.B. SSD-Pool für performancekritische VMs).

---

## Schritt 5: Pool konfigurieren

Der Pool definiert Replikationsfaktor und Mindest-Verfügbarkeit:

```bash
# VM-Storage-Pool erstellen und direkt als Proxmox-Storage registrieren
pveceph pool create vm-store \
  --size 3 \
  --min_size 2 \
  --pg_autoscale_mode on \
  --add_storages 1

# Optional: Container-Pool (gleiche Einstellungen)
pveceph pool create ct-store \
  --size 3 \
  --min_size 2 \
  --pg_autoscale_mode on \
  --add_storages 1
```

**Parameter-Erklärung:**

| Parameter | Wert | Bedeutung |
|-----------|------|-----------|
| `--size 3` | 3 Replikas | Jeder Block auf 3 verschiedenen Nodes |
| `--min_size 2` | Minimum 2 | Cluster bleibt schreibbar wenn 1 OSD/Node ausfällt |
| `--pg_autoscale_mode on` | Auto | Ceph verwaltet PG-Anzahl automatisch |
| `--add_storages 1` | Ja | Legt gleichzeitig Proxmox-Storage-Eintrag an |

> ⚠️ `min_size 1` niemals setzen — dann akzeptiert Ceph Schreibvorgänge ohne Replikation → Datenverlustrisiko bei OSD-Ausfall.

---

## Schritt 6: CRUSH-Regel prüfen

Die CRUSH-Regel definiert, auf welcher Ebene Daten redundant verteilt werden. Standard für 3-Node: **Host-Ebene** (keine zwei Replikas auf demselben Node):

```bash
# Aktive CRUSH-Regeln anzeigen
ceph osd crush rule ls

# Regel "replicated_rule" prüfen (Standard, host-Failure-Domain)
ceph osd crush rule dump replicated_rule

# Zugewiesene Regel für Pool prüfen
ceph osd pool get vm-store crush_rule
```

Korrekte CRUSH-Regel sicherstellt: Bei Verlust eines kompletten Nodes (alle 4 OSDs) bleiben immer noch 2 Replikas auf den anderen zwei Nodes verfügbar → `min_size=2` greift, Cluster bleibt verfügbar.

---

## Schritt 7: Proxmox-Storage verifizieren

Nach `--add_storages 1` ist der Pool bereits als Proxmox-Storage registriert. Prüfung:

```bash
# Storage-Liste
pvesm status

# Erwartete Ausgabe enthält:
# vm-store   rbd    active   <gesamt>   <belegt>   <verfügbar>
```

**Web UI:** Datacenter → Storage → `vm-store` sollte sichtbar und aktiv sein.

---

## Optonal: CephFS für ISO-Images & Shared Configs

CephFS bietet ein verteiltes Dateisystem auf Ceph-Basis. Nützlich für:
- ISO-Image-Repository (cluster-weit zugreifbar)
- Shared LXC-Configs

```bash
# CephFS erstellen (inkl. Metadata-Pool und Daten-Pool)
pveceph fs create --name cephfs --add-storage 1

# MDS-Daemon auf zwei Nodes (1 aktiv, 1 standby)
pveceph mds create
ssh root@pve2 "pveceph mds create"

# Status
ceph fs status
```

---

## Ceph Health Monitoring

```bash
# Cluster-Gesamtstatus
ceph status
ceph health detail

# OSD-Status aller 12 OSDs
ceph osd stat
ceph osd tree

# Pool-Nutzung
ceph df

# Performance (I/O-Rate)
ceph osd perf

# PG-Status (sollte "active+clean" sein)
ceph pg stat

# Echtzeit-I/O-Monitor
ceph -w
```

**Gewünschter Zielstatus:**

```
health: HEALTH_OK
  cluster:
    id: ...
    health: HEALTH_OK
  services:
    mon: 3 daemons, quorum pve1,pve2,pve3
    mgr: pve1(active), pve2(standby)
    osd: 12 osds: 12 up, 12 in
  data:
    pools: 2 pools, ...
    objects: ...
    pgs: ... active+clean
```

---

## CRUSH-basiertes Ausfallszenario

| Szenario | Ceph-Reaktion | Cluster-Status |
|----------|---------------|----------------|
| 1 OSD fällt aus | Daten werden auf verbleibende OSDs re-repliziert | HEALTH_WARN → nach Re-Replikation HEALTH_OK |
| 1 Node komplett aus (4 OSDs) | `min_size=2`: Cluster bleibt schreibbar | HEALTH_WARN, VM-Betrieb weiter |
| 2 Nodes aus (8 OSDs) | `min_size=2` nicht mehr erfüllbar → I/O gestoppt | HEALTH_ERR, VMs eingefroren |
| 1 Node aus, HA aktiv | HA-Manager startet VMs auf verbliebenen Nodes | Automatisches Failover (Kapitel 10) |

---

## Best Practices

✅ **Empfohlen:**
- Enterprise-SSDs mit PLP (Power Loss Protection) für OSD-Disks in Produktivumgebungen
- Separate WAL/DB-Disks (NVMe) bei HDD-OSDs für bessere Performance
- `pg_autoscale_mode on` – Ceph verwaltet PG-Anzahl selbst
- `min_size=2`, `size=3` für 3-Node-Cluster – niemals abweichen
- Regelmäßige `ceph health detail` prüfen via NinjaOne-Monitoring (Kapitel 06)
- Log-Rotation für Ceph-Logs sicherstellen (`/var/log/ceph/`)

⚠️ **Achtung:**
- Keine OS-Disks als OSD verwenden
- Hardware-RAID-Controller inkompatibel mit Ceph → HBA im IT-Mode erforderlich
- Bei `HEALTH_WARN` nie voreilig OSDs entfernen – Rebalancing abwarten
- `min_size 1` absolut vermeiden

❌ **Vermeiden:**
- Ceph auf < 3 Nodes betreiben
- Public und Cluster Network über Management-Interface (172.30.7.x) laufen lassen
- Ceph-OSD-Disks für andere Zwecke mitnutzen (z.B. LVM-Volumes)

---

## Häufige Fehler & Lösungen

| Problem | Mögliche Ursache | Lösung |
|---------|-----------------|--------|
| MON nicht im Quorum | Ceph-Netz nicht erreichbar | `ceph mon stat`; bond1-Konnektivität prüfen (`ping 10.20.20.12`) |
| OSD bleibt `down` | Disk-Fehler oder Berechtigungsproblem | `ceph osd metadata <id>`; `journalctl -u ceph-osd@<id>` |
| `HEALTH_WARN: slow ops` | HDD-Latenz, Netzwerk-Engpass | `ceph osd perf`; Bond1-Traffic prüfen |
| Pool `vm-store` fehlt in pvesm | `--add_storages` nicht gesetzt | `pvesm add rbd vm-store --pool vm-store --content images,rootdir` |
| `PGs inactive` | Zu wenige OSDs für min_size | Alle OSDs auf `up, in` bringen; `ceph osd in <id>` |
| `mon pve2 low on space` | `/var/log/ceph/` zu groß | `logrotate -f /etc/logrotate.d/ceph-common`; alte Logs bereinigen |

### Debugging-Befehle

```bash
# Ausführlicher Cluster-Status
ceph health detail

# OSD-Logs eines bestimmten OSDs
journalctl -u ceph-osd@0 --since "10 minutes ago"

# Monitor-Logs
journalctl -u ceph-mon@pve1 --since "1 hour ago"

# Ceph-Konfiguration anzeigen
ceph config dump

# PG-Details bei Problemen
ceph pg dump_stuck

# OSD manuell als "in" markieren (nach Wiederherstellung)
ceph osd in <osd-id>
```

---

## Weiterführende Links

- [Proxmox Ceph – Offizielle Dokumentation](https://pve.proxmox.com/wiki/Deploy_Hyper-Converged_Ceph_Cluster)
- [Ceph Dokumentation – Squid (19.x)](https://docs.ceph.com/en/squid/)
- [Kapitel 02 – Netzwerk & Clustering](02-netzwerk-clustering.md)
- [Kapitel 10 – High Availability](10-high-availability.md)
