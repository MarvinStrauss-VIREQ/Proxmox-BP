<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: Proxmox -->
<!-- Title: Proxmox - Monitoring & Alerting -->
<!-- Label: pve-monitoring, best-practice, checkliste -->

# Monitoring & Alerting

> **PVE 9.x / NinjaOne RMM** · Letzte Aktualisierung: 2026-06
> Quelle: adaptiert aus Kapitel 06 – Monitoring & Alerting.
> Verwandte Seiten: [Post installation check](installationen/post-installation-check.md) · [Sicherheit & Hardening](sicherheit-hardening.md)

---

## Wann anwenden?

- NinjaOne Agent auf PVE- und PBS-Nodes installieren
- Monitoring-Policy für Proxmox-Nodes konfigurieren (CPU, RAM, Disk, Services)
- Proxmox-spezifische Daten (Cluster-Status, VMs, Storage, ZFS) in NinjaOne Custom Fields schreiben
- Native PVE Notifications für Backup- und S.M.A.R.T.-Events einrichten (ergänzend zu NinjaOne)

---

## Überblick: Two-Layer-Ansatz

| Layer | Tool | Abgedeckt |
|---|---|---|
| Layer 1 – RMM | NinjaOne Agent + Policies | CPU, RAM, Disk, Services, Erreichbarkeit, Scripted Checks |
| Layer 2 – Native | PVE Notification System | Backup-Fehler, S.M.A.R.T.-Warnungen, ZFS-Events, Cluster-Events |

> **Warum beide?** NinjaOne überwacht den Proxmox-Host als Linux-Server. Proxmox-interne Events (z. B. ein fehlgeschlagener PBS-Backup-Job) kennt NinjaOne nur über eigene Skripte – das native PVE Notification System schließt diese Lücke direkt.

---

## 1. NinjaOne Agent auf PVE-Nodes installieren

> ⚠️ Proxmox 9 (Debian 13 Trixie) fällt unter „Extended Support“ in NinjaOne (nicht vollständig validiert, aber funktional).

```bash
# Installer herunterladen (Link aus NinjaOne-Konsole):
wget -O ninja-agent.deb "https://app.ninjarmm.com/agent/installer/..."
dpkg -i ninja-agent.deb
systemctl status ninjarmm-agent
```

---

## 2. NinjaOne Policy für Proxmox-Nodes

### Standard-Conditions

```
CPU-Last:    > 90 % für 15 Min → Warning, > 95 % → Critical
RAM:         > 85 % → Warning
Disk Root:   < 15 % frei → Warning, < 5 % → Critical
```

### Service-Conditions (kritische PVE-Dienste)

```
pveproxy       → Status: Running (Web-UI)
pvedaemon      → Status: Running (PVE-Core)
corosync       → Status: Running (nur bei Cluster-Nodes!)
ninjarmm-agent → Status: Running
```

> ⚠️ `corosync` nur auf Cluster-Nodes überwachen – auf Single-Nodes läuft der Dienst nicht und erzeugt Fehlalarm.

---

## 3. Proxmox-spezifisches Monitoring via NinjaOne Script Hub

NinjaOne bietet offizielle Bash-Skripte für Proxmox im Script Hub, die Daten via `ninjarmm-cli` in Custom Fields schreiben:

| Custom Field | Script Hub | Inhalt |
|---|---|---|
| `proxmox_cluster_status` | "Proxmox Cluster Status" | `pvecm status`-Ergebnis |
| `proxmox_node_status` | "Proxmox Node Monitoring" | Node-ID, Name, IP, Online-Status |
| `proxmox_vm_info` | "Proxmox VM Information Gathering" | VM-ID, Name, Status, RAM, CPU |
| `proxmox_storage_status` | "Retrieve Proxmox Node Storage Status" | Storage-Name, Typ, Größe, frei |

---

## 4. ZFS- und Quorum-Checks als Script Condition

NinjaOne hat keine native ZFS-Erkennung – dafür Custom Script Conditions:

```bash
#!/bin/bash
# zfs_health_check.sh – Exit 0 = OK, Exit 1 = Warning/Critical
DEGRADED=$(zpool list -H -o health | grep -cv "ONLINE")
if [ "$DEGRADED" -gt 0 ]; then
    echo "ZFS ALERT: $DEGRADED Pool(s) nicht ONLINE"
    zpool status | grep -E "state:|status:|pool:"
    exit 1
fi
echo "ZFS OK: Alle Pools ONLINE"
exit 0
```

```bash
#!/bin/bash
# corosync_quorum_check.sh – nur auf Cluster-Nodes
if ! command -v pvecm &>/dev/null; then
    echo "Kein Cluster-Node – Skript übersprungen"
    exit 0
fi
QUORATE=$(corosync-quorumtool -p 2>/dev/null | awk '/Quorate/ {print $2}')
if [ "$QUORATE" != "Yes" ]; then
    echo "CRITICAL: Cluster hat kein Quorum! ($QUORATE)"
    exit 1
fi
echo "OK: Cluster quorate"
exit 0
```

Beide als Script Condition mit Schedule "alle 15 Minuten", Trigger "Exit Code != 0", Severity Critical konfigurieren.

---

## 5. Native PVE Notifications (ergänzend)

```
PVE Web-UI: Datacenter → Notifications → Add → SMTP
Host: smtp.office365.com, Port: 587, Mode: STARTTLS
Username: proxmox-alert@vireq.com
```

```
Notification Rules → Add
Regel 1 – Backup-Fehler (PBS): severity >= error, source = backup
Regel 2 – S.M.A.R.T.-Fehler: severity >= warning, source = system
Regel 3 – Cluster-Events: severity >= error, source = cluster
```

> 💡 PBS-Backup-Fehler und S.M.A.R.T.-Events entstehen innerhalb von Proxmox und sind für NinjaOne ohne aufwändige Skripte nicht sofort sichtbar – die native PVE Notification schlägt schneller an.

---

## 6. Alert-Schwellwerte Referenz

| Alarm | Schwellwert | Typ | Priorität |
|---|---|---|---|
| CPU-Last | > 90 % für 15 Min | Condition | Warning |
| RAM | > 85 % | Condition | Warning |
| Disk Root | < 15 % frei | Condition | Warning |
| pveproxy/pvedaemon/corosync gestoppt | Status != Running | Service Condition | Critical |
| ZFS Pool degraded | Script Exit 1 | Script Condition | Critical |
| Cluster kein Quorum | Script Exit 1 | Script Condition | Critical |
| Backup-Job fehlgeschlagen | > 24h kein Backup | Native PVE Notify | Critical |
| S.M.A.R.T.-Fehler | Reallocated Sectors > 0 | Native PVE Notify | Critical |

---

## Best Practices

✅ NinjaOne Policy separat für Proxmox-Nodes – nicht mit Windows-Server-Policies mischen
✅ Custom Fields für Cluster-/Node-/VM-Status täglich automatisch befüllen
✅ ZFS- und Quorum-Checks alle 15 Minuten als Script Condition
✅ Native PVE Notifications für Backup und S.M.A.R.T. immer aktiv halten

⚠️ Script-Conditions laufen mit Root-Rechten – Skripte vor Deployment prüfen
⚠️ `corosync`-Condition nur auf Cluster-Nodes aktiv setzen

❌ Separate Grafana/InfluxDB-Instanz aufbauen, wenn NinjaOne bereits im Einsatz ist
❌ Monitoring-User mit Admin-Rechten in PVE – `PVEAuditor` reicht

---

## Häufige Fehler & Lösungen

**NinjaOne Agent erscheint nicht in der Konsole**
```
Prüfen:  systemctl status ninjarmm-agent
Ursache: Firewall blockiert ausgehende Verbindung (Port 443)
Lösung:  Ausgehenden HTTPS-Traffic zu *.ninjarmm.com erlauben
```

**E-Mail-Benachrichtigungen kommen nicht an**
```
Prüfen:  echo "Test" | mail -s "PVE Test" technik@vireq.com; mailq; journalctl -u postfix
```

---

## Weiterführende Links

| Ressource | URL |
|---|---|
| NinjaOne Linux Agent Installation | https://www.ninjaone.com/docs/new-to-ninjaone/agent-installation/linux-device-agent-installation/ |
| PVE Notification System | https://pve.proxmox.com/wiki/Notifications |
