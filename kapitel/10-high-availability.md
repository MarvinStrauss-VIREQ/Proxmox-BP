<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: PVE – Best Practices -->
<!-- Title: PVE – High Availability: HA Manager, Fencing & Ressourcen -->
<!-- Label: pve-cluster, best-practice, referenz, pve-sicherheit -->

# Kapitel 10: High Availability (HA)

> **PVE 9.x (Debian 13 Trixie)** · Letzte Aktualisierung: 2026-06  
> Verwandte Kapitel: [02 – Netzwerk & Clustering](02-netzwerk-clustering.md) · [09 – Ceph Storage](09-ceph-storage.md)

---

## Wann anwenden?

- **Produktiv-VMs mit SLA** – VMs die bei Node-Ausfall automatisch auf anderen Nodes neu starten sollen
- **3-Node-Cluster mit Ceph** – Shared Storage (Kapitel 09) ist Voraussetzung
- **Maximum Ausfallsicherheit** – Ergänzt Ceph (Storage-Redundanz) um die Compute-Redundanz

> **Nicht für Test-/Dev-VMs** – HA bedeutet Overhead durch Fencing-Timeouts und Migrationsvorgänge. Nur produktionskritische Ressourcen unter HA stellen.

---

## Überblick & Failover-Ablauf

Der Proxmox HA Manager erkennt automatisch Ausfälle und startet betroffene VMs auf verfügbaren Nodes neu — vollständig automatisch, ohne manuellen Eingriff.

```
Normalbetrieb:
  pve1 (VM 100, 101)    pve2 (VM 102, 103)    pve3 (VM 104, 105)
  HA-LRM aktiv          HA-LRM aktiv           HA-LRM aktiv
  HA-CRM: pve1 (Master)

──── pve1 fällt aus ─────────────────────────────────────────────

  1. Corosync erkennt pve1 unerreichbar (< 30 Sekunden)
  2. pve2+pve3 behalten Quorum (2 von 3 Votes)
  3. HA-CRM wechselt auf pve2
  4. FENCING: Watchdog auf pve1 hat gefeuert → pve1 rebooted
  5. HA-Manager bestätigt: pve1 ist fenced (Heartbeat weg)
  6. VM 100 + VM 101 → Start auf pve2 oder pve3 (je nach Gruppe)
  7. Gesamtzeit: ~60–120 Sekunden (Fencing-Timeout + VM-Start)
```

### Hyper-V Vergleich

| Konzept | Windows Server Failover Cluster (WSFC) | Proxmox HA Manager |
|---------|---------------------------------------|-------------------|
| Cluster-Mechanismus | Windows Clustering (AD-abhängig) | Corosync + pmxcfs (AD-unabhängig) |
| Shared Storage | SAN / CSV / S2D erforderlich | Ceph RBD (nativ) |
| Fencing | STONITH / Witness-Disk / Cloud-Witness | Watchdog + optionales IPMI |
| Failover-Erkennung | ~30 Sekunden | ~30 Sekunden |
| VM-Neustart | Automatisch | Automatisch |
| Failback | Konfigurierbar | Konfigurierbar (`nofailback`) |
| Konfiguration | Failover-Cluster-Manager (GUI) | Datacenter → HA (GUI + CLI) |

---

## HA-Architektur

```
Cluster Resource Manager (CRM)     ← Läuft auf einem Node (Master)
  └── Entscheidet, WO Ressourcen laufen
  └── Initiiert Fencing bei Node-Ausfall
  └── Koordiniert Migrations- und Failover-Aktionen

Local Resource Manager (LRM)       ← Läuft auf jedem Node
  └── Startet/stoppt Ressourcen auf dem lokalen Node
  └── Setzt CRM-Entscheidungen um

Watchdog                           ← Läuft auf jedem Node
  └── Hardware-Watchdog (/dev/watchdog0) oder Softdog
  └── Node der Quorum verliert → Watchdog feuert → Reboot
  └── Das ist Fencing: Node beendet sich selbst
```

---

## Voraussetzungen-Checkliste

Vor HA-Aktivierung **alle Punkte verifizieren**:

```bash
# 1. Quorum intakt (alle 3 Nodes online)
pvecm status
# → "Quorate: Yes", "Votes: 3"

# 2. Ceph Storage verfügbar (Shared Storage Pflicht)
pvesm status | grep vm-store
# → vm-store  rbd  active ...

# 3. Watchdog-Device verfügbar (Fencing-Voraussetzung)
ls -la /dev/watchdog*
# → Erwartet: /dev/watchdog oder /dev/watchdog0

# 4. HA-Dienste aktiv
systemctl status pve-ha-crm pve-ha-lrm

# 5. Ceph HEALTH_OK (keine Degradation vor HA-Tests)
ceph health
```

---

## Schritt 1: Watchdog einrichten (Fencing)

Der Watchdog ist die **primäre Fencing-Methode** in Proxmox. Ein Node der seinen Cluster-Heartbeat verliert, wird nicht länger vom HA-Manager als "lebendig" gewertet — der Watchdog feuert nach Timeout und erzwingt einen Reboot. Erst dann darf der HA-Manager die VMs des gefallenen Nodes auf anderen Nodes starten.

**Warum Fencing zwingend nötig ist:** Ohne Fencing könnte der HA-Manager VMs auf anderen Nodes starten während die ursprüngliche Node noch läuft → Split-Brain → Datenkorrumpierung auf Ceph.

```bash
# Hardware-Watchdog prüfen (bevorzugt)
ls /dev/watchdog*
dmesg | grep -i watchdog

# Falls kein Hardware-Watchdog: Softdog laden
modprobe softdog
echo "softdog" >> /etc/modules

# Watchdog-Timeout prüfen (Standard: 60 Sekunden)
cat /sys/class/watchdog/watchdog0/timeout

# HA-Manager Watchdog-Nutzung bestätigen
journalctl -u pve-ha-lrm | grep -i watchdog
```

**Auf allen drei Nodes ausführen.**

---

## Schritt 2: IPMI Fence Device einrichten (empfohlen)

Das IPMI-Fence-Device ist ein **zusätzlicher Sicherheitsmechanismus**: Falls der Watchdog eines Nodes nicht feuert (z.B. Kernel-Freeze), kann der HA-Manager den Node aktiv über IPMI ausschalten.

**Web UI:** Datacenter → HA → Fencing → Add

| Feld | Wert (Beispiel pve1) |
|------|---------------------|
| Fence Type | `IPMI` |
| Node | `pve1` |
| IPMI Host | IP des iDRAC/BMC von pve1 |
| User | IPMI-User (z.B. `root`) |
| Password | IPMI-Passwort (in Bitwarden speichern!) |

Für alle drei Nodes wiederholen. IPMI-IPs sind die Out-of-Band-Management-Adressen (unabhängig vom regulären Netzwerk).

```bash
# IPMI-Erreichbarkeit testen (von pve2 auf pve1 IPMI)
ipmitool -I lanplus -H <ipmi-ip-pve1> -U root -P <password> power status
# Erwartet: "Chassis Power is on"

# Manueller Fence-Test (VORSICHT: pve1 wird ausgeschaltet!)
# Nur im Wartungsfenster testen
# ipmitool -I lanplus -H <ipmi-ip-pve1> -U root -P <password> power off
```

---

## Schritt 3: HA-Gruppe erstellen

HA-Gruppen definieren, **auf welchen Nodes** HA-Ressourcen laufen dürfen und mit welcher Priorität.

**Web UI:** Datacenter → HA → Groups → Add

| Feld | Wert | Erklärung |
|------|------|-----------|
| ID | `ha-main` | Name der Gruppe |
| Nodes | `pve1:3,pve2:2,pve3:1` | Priorität pro Node (höher = bevorzugt) |
| Nofailback | `0` (deaktiviert) | VM kehrt nach Node-Wiederherstellung zurück |
| Restricted | `0` (deaktiviert) | VM kann auf alle Nodes migrieren |

**Per CLI:**
```bash
ha-manager groupadd ha-main \
  --nodes pve1:3,pve2:2,pve3:1 \
  --nofailback 0 \
  --restricted 0
```

> **Nofailback:** Bei `nofailback=0` migriert die VM nach Node-Wiederherstellung zurück auf den bevorzugten Node. Bei kritischen produktiven VMs kann `nofailback=1` sinnvoll sein um automatische Migrationen zu minimieren.

---

## Schritt 4: VM-Voraussetzungen für HA

Bevor VMs unter HA-Kontrolle gestellt werden, müssen sie korrekt konfiguriert sein:

### 4a: QEMU Guest Agent aktivieren

Der Guest Agent ermöglicht ordnungsgemäßes Shutdown bei Migrationen. Ohne ihn sendet Proxmox nur ein ACPI-Signal.

```bash
# VM-Konfiguration (auf dem Proxmox-Host)
qm set <vmid> --agent enabled=1,fstrim_cloned_disks=1

# Im Gast-OS installieren (Debian/Ubuntu)
apt install qemu-guest-agent
systemctl enable --now qemu-guest-agent

# Im Gast-OS (Windows)
# Virtio-Treiber installieren → Guest Agent läuft automatisch
# Download: https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/

# Agent-Status vom Host aus prüfen
qm agent <vmid> ping
```

### 4b: CPU-Typ für Live-Migration

Bei homogenen Clustern (identische Hardware) ist `host` optimal. Für Cluster mit unterschiedlichen CPU-Generationen:

```bash
# Migrationssicherer CPU-Typ setzen
qm set <vmid> --cpu x86-64-v2-AES

# Aktuellen CPU-Typ prüfen
qm config <vmid> | grep cpu
```

| CPU-Typ | Performance | Migration | Empfehlung |
|---------|-------------|-----------|------------|
| `host` | ⭐⭐⭐ Maximal | Nur identische CPUs | Homogene Cluster |
| `x86-64-v3` | ⭐⭐ Hoch | Intel Haswell+ / AMD Zen+ | Gemischte moderne Cluster |
| `x86-64-v2-AES` | ⭐ Standard | Alle x86-64 mit AES-NI | Sicherster Wert für Migration |
| `kvm64` | ⭐ Basis | Universell | Legacy / maximale Kompatibilität |

### 4c: Disk auf Shared Storage

HA-VMs **müssen** ihre Disks auf Ceph RBD (oder anderem Shared Storage) haben — lokaler Storage ist nicht migrierbar:

```bash
# Disk von lokalem Storage auf Ceph migrieren
qm move-disk <vmid> scsi0 vm-store --delete 1

# Storage-Typ der Disk prüfen
qm config <vmid> | grep scsi
# Soll enthalten: vm-store (RBD), nicht local-lvm oder local
```

---

## Schritt 5: VMs unter HA-Kontrolle stellen

**Web UI:** Datacenter → HA → Add (Resource)

| Feld | Wert | Erklärung |
|------|------|-----------|
| VM ID | `100` | VM-ID |
| Group | `ha-main` | Vorher angelegte Gruppe |
| State | `started` | Gewünschter Betriebszustand |
| Max. Restart | `3` | Maximale Neustart-Versuche bei Crash |
| Max. Relocate | `3` | Maximale Migrations-Versuche bei Node-Ausfall |

**Per CLI:**
```bash
# Einzelne VM unter HA-Kontrolle
ha-manager add vm:100 \
  --group ha-main \
  --state started \
  --max_restart 3 \
  --max_relocate 3

# Weitere VMs
ha-manager add vm:101 --group ha-main --state started --max_restart 3 --max_relocate 3
ha-manager add vm:102 --group ha-main --state started --max_restart 3 --max_relocate 3

# Alle HA-Ressourcen anzeigen
ha-manager status
```

---

## Schritt 6: Vollständige Gesamtarchitektur

```
3-Node HA Cluster (Maximum Ausfallsicherheit)
│
├── Layer 1: Netzwerk (Kapitel 02, 08)
│   ├── Management + Corosync Ring 1  →  172.30.7.x/16   (vmbr0 / nic0)
│   ├── Corosync Ring 0 (dediziert)   →  10.10.20.x/24   (ens19)
│   ├── Ceph-Netz                     →  10.20.20.x/24   (bond1)
│   └── VM-Traffic (SDN VLAN Zone)    →  bond0 → vmbr1 → VNets
│
├── Layer 2: Cluster & Quorum (Kapitel 02)
│   ├── Corosync (2 Ringe, redundant)
│   ├── 3 Nodes = natürliches Quorum (kein QDevice nötig)
│   └── pmxcfs (verteiltes Cluster-Dateisystem)
│
├── Layer 3: Shared Storage / Ceph (Kapitel 09)
│   ├── 12 OSDs (4 pro Node, HDD/SAS)
│   ├── Pool vm-store  →  size=3, min_size=2, CRUSH: host-Ebene
│   └── Übersteht 1 Node-Komplettausfall ohne Datenverlust
│
├── Layer 4: High Availability (dieses Kapitel)
│   ├── Watchdog Fencing  →  Node ohne Quorum rebooted sich selbst
│   ├── IPMI Fencing      →  Aktives Ausschalten als Backup
│   ├── HA Gruppe: ha-main  →  pve1:3, pve2:2, pve3:1
│   └── HA Ressourcen  →  VMs mit max_restart=3, max_relocate=3
│
├── Layer 5: Sicherheit & Firewall (Kapitel 05, 07)
│   ├── cluster.fw (DROP policy, 3 IPsets)
│   └── SSH-Härtung, 2FA, TLS-Zertifikate
│
└── Layer 6: Backup & Monitoring (Kapitel 03, 06)
    ├── PBS (Backup Server, unabhängig vom Cluster)
    ├── Hornetsecurity VM Backup (ab PVE 9.0+)
    └── NinjaOne Monitoring (Kapitel 06)
```

---

## Schritt 7: HA-Failover testen

**Immer im Wartungsfenster testen!** Empfohlene Testreihenfolge:

### Test 1: Geplante Migration (kein Ausfall)

```bash
# VM 100 von pve1 auf pve2 live migrieren
qm migrate 100 pve2 --online 1

# Status beobachten
qm status 100
```

### Test 2: Node-Shutdown (kontrollierter Ausfall)

```bash
# pve3 in Wartungsmodus versetzen (VMs werden migriert)
ha-manager crm-command node-maintenance enable pve3

# HA-Status beobachten
watch ha-manager status

# Alle VMs sollen auf pve1 oder pve2 landen
# Nach Test: Wartungsmodus beenden
ha-manager crm-command node-maintenance disable pve3
```

### Test 3: Ungeplanter Ausfall (Worst Case)

```bash
# ACHTUNG: Nur im Testbetrieb ohne produktive Last!
# Simuliert harten Node-Absturz auf pve1

# Von pve2 aus: pve1 über IPMI hart ausschalten
ipmitool -I lanplus -H <ipmi-ip-pve1> -U root -P <password> power off

# HA-Status beobachten (von pve2 oder pve3)
watch ha-manager status

# Erwarteter Ablauf:
# 1. Corosync erkennt pve1 verloren (~10–30 Sek.)
# 2. Fencing: Watchdog/IPMI bestätigt pve1 aus
# 3. VMs von pve1 starten auf pve2/pve3 (~60–120 Sek. gesamt)

# pve1 wieder einschalten nach Test
ipmitool -I lanplus -H <ipmi-ip-pve1> -U root -P <password> power on
```

---

## HA-Status Monitoring

```bash
# Gesamtstatus aller HA-Ressourcen
ha-manager status

# Erwartete Ausgabe bei Normalbetrieb:
# quorum OK
# master pve1
# lrm pve1: online
# lrm pve2: online
# lrm pve3: online
# service vm:100 (pve1) : started
# service vm:101 (pve2) : started

# Echtzeit-Statusüberwachung
watch -n 2 ha-manager status

# HA-CRM-Log (Entscheidungen des Cluster Resource Managers)
journalctl -u pve-ha-crm --since "1 hour ago"

# HA-LRM-Log (Aktionen auf lokalem Node)
journalctl -u pve-ha-lrm --since "1 hour ago"

# Fencing-Events in Systemlog
journalctl -k | grep -i "watchdog\|fence\|reboot"
```

---

## HA-Konfigurationsdateien

| Datei | Inhalt |
|-------|--------|
| `/etc/pve/ha/crm_commands` | Aktuelle CRM-Befehle |
| `/etc/pve/ha/groups.cfg` | HA-Gruppen-Definitionen |
| `/etc/pve/ha/resources.cfg` | HA-Ressourcen-Definitionen |
| `/etc/pve/ha/fence.cfg` | Fencing-Device-Konfiguration |

Alle Dateien werden durch pmxcfs cluster-weit synchronisiert.

---

## Ausfallszenarien & Erwartetes Verhalten

| Szenario | Quorum | Ceph | HA-Reaktion | Downtime |
|----------|--------|------|-------------|---------|
| 1 Node komplett aus | ✅ 2 Votes verbleiben | ✅ min_size=2 ok | VMs automatisch neu gestartet | ~60–120 Sek. |
| Netzwerkausfall 1 Node | ✅ 2 Votes verbleiben | ✅ I/O ok | Fencing → Node reboots → VM-Failover | ~60–120 Sek. |
| 2 Nodes gleichzeitig aus | ❌ Quorum verloren | ❌ I/O gestoppt | Kein HA-Failover möglich | Bis Nodes wieder online |
| Node-Reboot (geplant) | ✅ | ✅ | VMs live-migriert (Wartungsmodus) | 0 Sek. (Live-Migration) |
| OSD-Ausfall (einzeln) | ✅ | ✅ Re-Replikation | Kein HA-Eingriff nötig | 0 Sek. |

---

## Best Practices

✅ **Empfohlen:**
- Immer QEMU Guest Agent in allen HA-VMs installieren
- CPU-Typ `x86-64-v2-AES` statt `host` für migrationssichere VMs
- IPMI-Fencing als zusätzliche Sicherheit neben Watchdog einrichten
- HA-Failover mindestens einmal im Quartal testen (Wartungsfenster)
- Backup (PBS) unabhängig von HA betreiben – HA schützt vor Node-Ausfall, nicht vor Datenverlust
- `max_restart=3` und `max_relocate=3` – verhindert Endlosschleife bei dauerhaft defekter VM

⚠️ **Achtung:**
- HA nur für VMs auf Shared Storage (Ceph RBD) aktivieren – lokale Disks sind nicht migrierbar
- Fencing-Timeout ist ~60 Sekunden – das ist normale Downtime bei ungeplanten Ausfällen
- `nofailback=0` bedeutet VMs migrieren zurück wenn bevorzugter Node wiederkommt → ggf. unerwünschte Migrationen in Produktivumgebungen

❌ **Vermeiden:**
- HA aktivieren ohne Watchdog (`/dev/watchdog` muss vorhanden sein)
- HA ohne Shared Storage (VMs auf local-lvm können nicht migriert werden)
- Test-VMs oder Build-VMs unter HA stellen (unnötiger Overhead)
- Beide HA-Manager-Dienste (CRM + LRM) manuell stoppen ohne `ha-manager` CLI

---

## Häufige Fehler & Lösungen

| Problem | Mögliche Ursache | Lösung |
|---------|-----------------|--------|
| VM startet nach Failover nicht | Disk auf local-lvm statt Ceph | `qm move-disk` auf vm-store; Ceph-Status prüfen |
| Fencing erfolgt nicht | Kein `/dev/watchdog` | `modprobe softdog`; `echo softdog >> /etc/modules` |
| HA-Status zeigt `error` | QEMU Guest Agent fehlt | Agent installieren; `qm set <vmid> --agent enabled=1` |
| VM migriert nicht (live) | CPU-Typ `host` bei inhomogenen Nodes | CPU-Typ auf `x86-64-v2-AES` ändern |
| `ha-manager status` zeigt `slave` | CRM auf anderem Node aktiv | Normal – CRM ist immer auf einem Node aktiv |
| Failover dauert >5 Minuten | IPMI-Fencing nicht konfiguriert | IPMI-Device hinzufügen; Watchdog-Timeout prüfen |

---

## Weiterführende Links

- [Proxmox HA Manager – Offizielle Dokumentation](https://pve.proxmox.com/wiki/High_Availability)
- [Proxmox Fencing – Dokumentation](https://pve.proxmox.com/wiki/Fencing)
- [Kapitel 09 – Ceph Storage](09-ceph-storage.md)
- [Kapitel 02 – Netzwerk & Clustering](02-netzwerk-clustering.md)
