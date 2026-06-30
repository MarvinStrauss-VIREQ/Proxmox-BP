<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: Proxmox -->
<!-- Title: Proxmox - High Availability -->
<!-- Label: pve-cluster, pve-sicherheit, best-practice, referenz -->

# High Availability (HA)

> **PVE 9.x (Debian 13 Trixie)** · Letzte Aktualisierung: 2026-06
> Quelle: adaptiert aus Kapitel 10 – High Availability.
> Verwandte Seiten: [Ceph Cluster - Netzwerk](netzwerk-firewall-konfigurationen/ceph-cluster-netzwerk.md) · [Ceph Storage](ceph-storage.md) · [Firewall](netzwerk-firewall-konfigurationen/firewall.md)

---

## Wann anwenden?

- Produktiv-VMs mit SLA – VMs, die bei Node-Ausfall automatisch auf anderen Nodes neu starten sollen
- 3-Node-Cluster mit Ceph – Shared Storage ist Voraussetzung
- Maximale Ausfallsicherheit – ergänzt Ceph (Storage-Redundanz) um die Compute-Redundanz

> **Nicht für Test-/Dev-VMs** – HA bedeutet Overhead durch Fencing-Timeouts und Migrationsvorgänge. Nur produktionskritische Ressourcen unter HA stellen.

---

## Überblick & Failover-Ablauf

Der Proxmox HA Manager erkennt Ausfälle automatisch und startet betroffene VMs auf verfügbaren Nodes neu – vollständig automatisch, ohne manuellen Eingriff.

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

### Hyper-V-Vergleich

| Konzept | Windows Server Failover Cluster (WSFC) | Proxmox HA Manager |
|---|---|---|
| Cluster-Mechanismus | Windows Clustering (AD-abhängig) | Corosync + pmxcfs (AD-unabhängig) |
| Shared Storage | SAN / CSV / S2D erforderlich | Ceph RBD (nativ) |
| Fencing | STONITH / Witness-Disk / Cloud-Witness | Watchdog + optionales IPMI |
| Failover-Erkennung | ~30 Sekunden | ~30 Sekunden |
| Konfiguration | Failover-Cluster-Manager (GUI) | Datacenter → HA (GUI + CLI) |

---

## HA-Architektur

```
Cluster Resource Manager (CRM)     ← Läuft auf einem Node (Master)
  └── Entscheidet, WO Ressourcen laufen, initiiert Fencing bei Node-Ausfall

Local Resource Manager (LRM)       ← Läuft auf jedem Node
  └── Startet/stoppt Ressourcen auf dem lokalen Node

Watchdog                           ← Läuft auf jedem Node
  └── Node der Quorum verliert → Watchdog feuert → Reboot (= Fencing)
```

---

## Voraussetzungen-Checkliste

```bash
# 1. Quorum intakt
pvecm status   # → "Quorate: Yes", "Votes: 3"

# 2. Ceph Storage verfügbar (Shared Storage Pflicht)
pvesm status | grep vm-store

# 3. Watchdog-Device verfügbar
ls -la /dev/watchdog*

# 4. HA-Dienste aktiv
systemctl status pve-ha-crm pve-ha-lrm

# 5. Ceph HEALTH_OK
ceph health
```

---

## Schritt 1: Watchdog einrichten (Fencing)

Der Watchdog ist die primäre Fencing-Methode. Ein Node, der seinen Cluster-Heartbeat verliert, wird nach Timeout zwangsweise neu gestartet – erst dann darf der HA-Manager dessen VMs anderswo starten. **Ohne Fencing droht Split-Brain und Datenkorruption auf Ceph.**

```bash
ls /dev/watchdog*
dmesg | grep -i watchdog

# Falls kein Hardware-Watchdog: Softdog laden
modprobe softdog
echo "softdog" >> /etc/modules

cat /sys/class/watchdog/watchdog0/timeout
journalctl -u pve-ha-lrm | grep -i watchdog
```

**Auf allen drei Nodes ausführen.**

---

## Schritt 2: IPMI Fence Device einrichten (empfohlen)

Zusätzlicher Sicherheitsmechanismus: Falls der Watchdog nicht feuert (z. B. Kernel-Freeze), kann der HA-Manager den Node aktiv über IPMI ausschalten.

**Web UI:** Datacenter → HA → Fencing → Add

> 📸 **Screenshot machen:** Datacenter → HA → Fencing → Add-Dialog mit ausgefüllten Feldern (Fence Type: IPMI, Node, IPMI Host, User) für einen eurer drei Nodes – zeigt Lesern auf einen Blick, wo das Fencing-Device konfiguriert wird.

| Feld | Wert (Beispiel pve1) |
|---|---|
| Fence Type | `IPMI` |
| Node | `pve1` |
| IPMI Host | IP des iDRAC/BMC von pve1 |
| Password | in Bitwarden hinterlegen |

```bash
# IPMI-Erreichbarkeit testen
ipmitool -I lanplus -H <ipmi-ip-pve1> -U root -P <password> power status
```

---

## Schritt 3: HA-Gruppe erstellen

Alternativ zur CLI auch über **Web UI: Datacenter → HA → Groups → Add** möglich.

> 📸 **Screenshot machen:** Datacenter → HA → Groups → Add-Dialog mit eurer `ha-main`-Gruppe (Nodes mit Priorität pve1:3, pve2:2, pve3:1) – zeigt die Node-Priorisierung visuell, die in der CLI nicht so eindeutig erkennbar ist.

```bash
ha-manager groupadd ha-main \
  --nodes pve1:3,pve2:2,pve3:1 \
  --nofailback 0 \
  --restricted 0
```

> `nofailback=0`: VM migriert nach Node-Wiederherstellung zurück auf den bevorzugten Node. Bei kritischen VMs kann `nofailback=1` automatische Rück-Migrationen vermeiden.

---

## Schritt 4: VM-Voraussetzungen für HA

### QEMU Guest Agent

```bash
qm set <vmid> --agent enabled=1,fstrim_cloned_disks=1
# Im Gast: apt install qemu-guest-agent && systemctl enable --now qemu-guest-agent
qm agent <vmid> ping
```

### CPU-Typ für Live-Migration

| CPU-Typ | Migration | Empfehlung |
|---|---|---|
| `host` | Nur identische CPUs | Homogene Cluster |
| `x86-64-v2-AES` | Alle x86-64 mit AES-NI | Sicherster Wert für Migration |

```bash
qm set <vmid> --cpu x86-64-v2-AES
```

### Disk auf Shared Storage

```bash
qm move-disk <vmid> scsi0 vm-store --delete 1
```

---

## Schritt 5: VMs unter HA-Kontrolle stellen

**Web UI:** Datacenter → HA → Add (Resource)

> 📸 **Screenshot machen:** Datacenter → HA → Add-Dialog für eine VM (VM ID, Group: ha-main, State: started, Max. Restart: 3, Max. Relocate: 3) – ideal direkt im Anschluss an einen echten Workflow bei einem Kundenprojekt.

```bash
ha-manager add vm:100 --group ha-main --state started --max_restart 3 --max_relocate 3
ha-manager status
```

---

## Schritt 6: HA-Failover testen

**Immer im Wartungsfenster testen!**

```bash
# Test 1: Geplante Live-Migration
qm migrate 100 pve2 --online 1

# Test 2: Node-Wartungsmodus (kontrollierter Ausfall)
ha-manager crm-command node-maintenance enable pve3
watch ha-manager status
ha-manager crm-command node-maintenance disable pve3

# Test 3: Ungeplanter Ausfall (nur ohne produktive Last!)
ipmitool -I lanplus -H <ipmi-ip-pve1> -U root -P <password> power off
watch ha-manager status
ipmitool -I lanplus -H <ipmi-ip-pve1> -U root -P <password> power on
```

---

## HA-Status Monitoring

Auch als Web UI verfügbar: **Datacenter → HA → Status**

> 📸 **Screenshot machen:** Datacenter → HA → Status-Übersicht während des Normalbetriebs (zeigt CRM-Master, LRM-Status aller Nodes, laufende HA-Ressourcen) – am besten direkt nach einem erfolgreichen Failover-Test, damit man den "vorher/nachher"-Zustand dokumentieren kann.

```bash
ha-manager status
watch -n 2 ha-manager status
journalctl -u pve-ha-crm --since "1 hour ago"
journalctl -u pve-ha-lrm --since "1 hour ago"
journalctl -k | grep -i "watchdog\|fence\|reboot"
```

---

## Ausfallszenarien & erwartetes Verhalten

| Szenario | Quorum | Ceph | HA-Reaktion | Downtime |
|---|---|---|---|---|
| 1 Node komplett aus | ✅ 2 Votes | ✅ min_size=2 ok | VMs automatisch neu gestartet | ~60–120 Sek. |
| 2 Nodes gleichzeitig aus | ❌ Quorum verloren | ❌ I/O gestoppt | Kein HA-Failover möglich | Bis Nodes online |
| Node-Reboot (geplant) | ✅ | ✅ | Live-Migration | 0 Sek. |

---

## Best Practices

✅ QEMU Guest Agent in allen HA-VMs installieren
✅ CPU-Typ `x86-64-v2-AES` statt `host` für migrationssichere VMs
✅ IPMI-Fencing zusätzlich zum Watchdog
✅ HA-Failover mindestens einmal im Quartal testen
✅ Backup (PBS) unabhängig von HA betreiben – HA schützt vor Node-Ausfall, nicht vor Datenverlust

⚠️ HA nur für VMs auf Shared Storage (Ceph RBD) – lokale Disks sind nicht migrierbar
⚠️ Fencing-Timeout ist ~60 Sekunden – normale Downtime bei ungeplanten Ausfällen

❌ HA aktivieren ohne Watchdog
❌ Test-/Build-VMs unter HA stellen

---

## Häufige Fehler & Lösungen

| Problem | Ursache | Lösung |
|---|---|---|
| VM startet nach Failover nicht | Disk auf local-lvm statt Ceph | `qm move-disk` auf vm-store |
| Fencing erfolgt nicht | Kein `/dev/watchdog` | `modprobe softdog` |
| HA-Status zeigt `error` | QEMU Guest Agent fehlt | Agent installieren, `--agent enabled=1` |
| VM migriert nicht (live) | CPU-Typ `host` bei inhomogenen Nodes | CPU-Typ auf `x86-64-v2-AES` ändern |

---

## Weiterführende Links

| Ressource | URL |
|---|---|
| Proxmox HA Manager | https://pve.proxmox.com/wiki/High_Availability |
| Proxmox Fencing | https://pve.proxmox.com/wiki/Fencing |
