<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: Proxmox -->
<!-- Title: Proxmox - Ceph Storage -->
<!-- Label: pve-storage, pve-cluster, best-practice, referenz -->

# Ceph Storage

> **PVE 9.x / Ceph 19.x Squid (Debian 13 Trixie)** · Letzte Aktualisierung: 2026-06
> Quelle: adaptiert aus Kapitel 09 – Ceph Storage. Deckt die Storage-Inbetriebnahme ab – das Netzwerk-Layout (NICs, Bonds, IP-Schema) steht in [Ceph Cluster - Netzwerk](netzwerk-firewall-konfigurationen/ceph-cluster-netzwerk.md).
> Verwandte Seiten: [High Availability](high-availability.md)

---

## Wann anwenden?

- **3-Node-Cluster** – Ceph ist der Standard für Shared Storage in Proxmox-Clustern ab 3 Nodes
- **Maximale Ausfallsicherheit** – Ceph mit `size=3` übersteht den Komplettausfall eines Nodes ohne Datenverlust
- **Live-Migration & HA** – Voraussetzung für VM-Live-Migration und HA-Manager (Shared Storage erforderlich)

> **Nicht für 2-Node-Cluster** – Ceph braucht mindestens 3 Nodes für ein sicheres Monitor-Quorum. Für 2 Nodes: NFS oder PBS als Storage.

---

## Überblick

Ceph ist ein verteiltes Software-Storage-System, das Redundanz direkt im Speicher-Layer verankert – ohne dedizierte Storage-Hardware. Jede VM-Disk wird in konfigurierten Replikas auf mehrere physische Nodes verteilt.

```
VM schreibt Daten → Ceph RBD → Ceph Pool (vm-store, size=3, CRUSH: host-Ebene)
  → Replica 1: OSD auf pve1   Replica 2: OSD auf pve2   Replica 3: OSD auf pve3
```

Fällt pve1 aus: Ceph arbeitet weiter mit den verbleibenden 2 Repliken (`min_size=2`) – Daten bleiben vollständig erhalten.

### Hyper-V-Vergleich

| Konzept | Hyper-V / Windows | Proxmox / Ceph |
|---|---|---|
| Shared Storage | SAN / iSCSI / SMB 3.0 | Ceph RBD (nativ im Cluster) |
| Redundanz | RAID-Controller, externes SAN | Software-RAID über alle Nodes |
| Ausfallsicherheit | Abhängig von SAN-Hardware | 1 Node-Ausfall bei size=3 ohne Datenverlust |
| Skalierung | Neue SAN-Hardware nötig | Node/OSD hinzufügen → automatisch balanciert |

> **Storage Spaces Direct (S2D)** ist der nächste Vergleichspunkt: ähnliches Prinzip wie Ceph, aber ausschließlich auf Windows beschränkt und lizenzpflichtig.

---

## Ceph-Architektur (3-Node-Cluster)

| Komponente | Anzahl | Beschreibung |
|---|---|---|
| MON (Monitor) | 3 (1 pro Node) | Cluster-Map, Quorum, Auth |
| MGR (Manager) | 2 (aktiv + standby) | Metriken, Dashboard, Module |
| OSD (Object Storage Daemon) | 12 (4 pro Node) | Speichert die tatsächlichen Daten |
| MDS (Metadata Server) | optional | Nur für CephFS nötig |

---

## Voraussetzungen

- 3 Nodes mit Corosync-Cluster, Ceph-Netz konfiguriert (→ [Ceph Cluster - Netzwerk](netzwerk-firewall-konfigurationen/ceph-cluster-netzwerk.md))
- Mindestens 1 dedizierte Disk pro OSD (keine OS-Disk!)
- Enterprise-SSDs mit PLP oder SAS-HDDs für Produktion
- Hardware-RAID-Controller im HBA-IT-Mode

```bash
pveceph status
lsblk -d -o NAME,SIZE,ROTA,TYPE | grep disk   # ROTA=1 = HDD, ROTA=0 = SSD/NVMe
```

---

## Schritt 1: Ceph initialisieren

```bash
# Auf pve1:
pveceph init --network 10.20.20.0/24
cat /etc/ceph/ceph.conf
```

> 📸 **Screenshot machen:** Datacenter → Ceph → Configuration-Übersicht direkt nach dem Init (zeigt Public/Cluster Network bereits korrekt auf 10.20.20.0/24 gesetzt) – wichtigster Kontrollpunkt, weil ein falsches Netz hier sich erst spät bemerkbar macht.

## Schritt 2: Monitore einrichten

Auch über **Web UI: Datacenter → Ceph → Monitor → Add** (auf jedem Node wiederholen).

> 📸 **Screenshot machen:** Datacenter → Ceph → Monitor-Tab mit allen 3 Monitoren im Status "running" – zeigt das Monitor-Quorum visuell, ohne dass man `ceph mon stat` in der Konsole lesen muss.

```bash
pveceph mon create
ssh root@pve2 "pveceph mon create"
ssh root@pve3 "pveceph mon create"
ceph mon stat   # quorum 0,1,2 pve1,pve2,pve3
```

## Schritt 3: Manager einrichten

```bash
pveceph mgr create
ssh root@pve2 "pveceph mgr create"
ceph mgr stat
```

## Schritt 4: OSDs erstellen

Auch über **Web UI: Node → Ceph → OSD → Create: OSD** möglich.

> 📸 **Screenshot machen:** Node → Ceph → OSD-Übersicht mit allen 4 OSDs eines Nodes im Status "up/in" inklusive CRUSH-Device-Class-Spalte (hdd/ssd) – gutes Referenzbild für "so soll es nach dem Setup aussehen".

```bash
pveceph osd create /dev/sdb --crush-device-class hdd
pveceph osd create /dev/sdc --crush-device-class hdd
ssh root@pve2 "pveceph osd create /dev/sdb --crush-device-class hdd"
# ... für alle Disks auf allen Nodes wiederholen
ceph osd tree
```

> `--crush-device-class hdd` klassifiziert die OSD korrekt – bei gemischten SSD/HDD-Setups ermöglicht das class-basierte CRUSH-Regeln.

## Schritt 5: Pool konfigurieren

Auch über **Web UI: Datacenter → Ceph → Pools → Create** möglich.

> 📸 **Screenshot machen:** Datacenter → Ceph → Pools → Create-Dialog mit `vm-store`, size=3, min_size=2, PG Autoscale Mode = on ausgefüllt – zeigt exakt die VIREQ-Standardwerte, die niemals abweichen sollen.

```bash
pveceph pool create vm-store \
  --size 3 --min_size 2 --pg_autoscale_mode on --add_storages 1
```

| Parameter | Wert | Bedeutung |
|---|---|---|
| `--size 3` | 3 Replikas | Jeder Block auf 3 verschiedenen Nodes |
| `--min_size 2` | Minimum 2 | Cluster bleibt schreibbar wenn 1 OSD/Node ausfällt |
| `--pg_autoscale_mode on` | Auto | Ceph verwaltet PG-Anzahl automatisch |

> ⚠️ `min_size 1` niemals setzen – dann akzeptiert Ceph Schreibvorgänge ohne Replikation.

## Schritt 6: CRUSH-Regel prüfen

```bash
ceph osd crush rule ls
ceph osd crush rule dump replicated_rule
ceph osd pool get vm-store crush_rule
```

## Schritt 7: Proxmox-Storage verifizieren

```bash
pvesm status   # vm-store sollte active erscheinen
```

---

## Optional: CephFS für ISO-Images & Shared Configs

```bash
pveceph fs create --name cephfs --add-storage 1
pveceph mds create
ssh root@pve2 "pveceph mds create"
ceph fs status
```

---

## Ceph Health Monitoring

```bash
ceph status
ceph health detail
ceph osd stat
ceph df
ceph osd perf
ceph pg stat
ceph -w
```

> 📸 **Screenshot machen:** Datacenter → Ceph → Hauptübersicht mit dem grünen "HEALTH_OK"-Status, der OSD-Map-Grafik und den Pool-Auslastungsbalken – das ist der Screenshot, der bei jedem Kunden-Health-Check als Erstes gezeigt wird.

**Gewünschter Zielstatus:** `health: HEALTH_OK`, alle Monitore in Quorum, alle OSDs `up, in`, alle PGs `active+clean`.

---

## CRUSH-basiertes Ausfallszenario

| Szenario | Ceph-Reaktion | Cluster-Status |
|---|---|---|
| 1 OSD fällt aus | Re-Replikation auf verbleibende OSDs | HEALTH_WARN → HEALTH_OK |
| 1 Node komplett aus (4 OSDs) | `min_size=2`: Cluster bleibt schreibbar | HEALTH_WARN, VM-Betrieb läuft weiter |
| 2 Nodes aus (8 OSDs) | `min_size=2` nicht mehr erfüllbar | HEALTH_ERR, VMs eingefroren |

---

## Best Practices

✅ Enterprise-SSDs mit PLP für OSD-Disks in Produktivumgebungen
✅ Separate WAL/DB-Disks (NVMe) bei HDD-OSDs für bessere Performance
✅ `pg_autoscale_mode on` – Ceph verwaltet PG-Anzahl selbst
✅ `min_size=2`, `size=3` für 3-Node-Cluster – niemals abweichen
✅ Regelmäßige `ceph health detail` prüfen via NinjaOne-Monitoring

⚠️ Keine OS-Disks als OSD verwenden
⚠️ Hardware-RAID-Controller inkompatibel mit Ceph → HBA im IT-Mode erforderlich
⚠️ Bei `HEALTH_WARN` nie voreilig OSDs entfernen – Rebalancing abwarten

❌ Ceph auf < 3 Nodes betreiben
❌ `min_size 1` setzen
❌ Ceph-OSD-Disks für andere Zwecke mitnutzen

---

## Häufige Fehler & Lösungen

| Problem | Ursache | Lösung |
|---|---|---|
| MON nicht im Quorum | Ceph-Netz nicht erreichbar | `ceph mon stat`; Netz-Konnektivität prüfen |
| OSD bleibt `down` | Disk-Fehler oder Berechtigungsproblem | `ceph osd metadata <id>`; `journalctl -u ceph-osd@<id>` |
| `HEALTH_WARN: slow ops` | HDD-Latenz, Netzwerk-Engpass | `ceph osd perf` |
| Pool fehlt in pvesm | `--add_storages` nicht gesetzt | `pvesm add rbd vm-store --pool vm-store --content images,rootdir` |

---

## Weiterführende Links

| Ressource | URL |
|---|---|
| Proxmox Ceph – Hyper-Converged Setup | https://pve.proxmox.com/wiki/Deploy_Hyper-Converged_Ceph_Cluster |
| Ceph Dokumentation – Squid (19.x) | https://docs.ceph.com/en/squid/ |
