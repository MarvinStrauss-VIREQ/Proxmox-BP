# Kapitel 6: Monitoring & Alerting

> **Version:** Proxmox VE 9.x / NinjaOne RMM · **Letzte Aktualisierung:** 2026-06  
> **Verwandte Kapitel:** [Kapitel 1](01-grundkonfiguration.md) (E-Mail/SMTP) · [Kapitel 5](05-sicherheit-hardening.md) (API-Token für Monitoring-User)

---

## Wann anwenden?

> - NinjaOne Agent auf PVE- und PBS-Nodes installieren
> - Monitoring-Policy für Proxmox-Nodes konfigurieren (CPU, RAM, Disk, Services)
> - Proxmox-spezifische Daten (Cluster-Status, VMs, Storage, ZFS) in NinjaOne Custom Fields schreiben
> - Native PVE Notifications für Backup- und SMART-Events einrichten (ergänzend zu NinjaOne)

---

## Überblick: Two-Layer-Ansatz

Proxmox-Monitoring mit NinjaOne funktioniert am besten zweischichtig:

| Layer | Tool | Abgedeckt |
|-------|------|-----------|
| **Layer 1 – RMM** | NinjaOne Agent + Policies | CPU, RAM, Disk, Services, Erreichbarkeit, Scripted Checks |
| **Layer 2 – Native** | PVE Notification System | Backup-Fehler, S.M.A.R.T.-Warnungen, ZFS-Events, Cluster-Events |

> **Warum beide?** NinjaOne überwacht den Proxmox-Host als Linux-Server. Proxmox-interne Events (z. B. ein fehlgeschlagener PBS-Backup-Job) kennt NinjaOne nur über eigene Skripte – das native PVE Notification System schließt diese Lücke direkt.

---

## 1. NinjaOne Agent auf PVE-Nodes installieren

NinjaOne unterstützt Linux-Systeme, die systemd verwenden – Proxmox VE basiert auf Debian und erfüllt diese Voraussetzung.

> ⚠️ **Proxmox 9 (Debian 13 Trixie):** Fällt unter „Extended Support" in NinjaOne (nicht vollständig validiert, aber funktional). Bei Problemen Support-Ticket öffnen.

### 1.1 Installer generieren

```
NinjaOne Konsole → (+) → Device → Linux
Organisation und Policy auswählen → Installer-Link generieren
```

### 1.2 Agent installieren (auf jedem PVE- und PBS-Node)

```bash
# Installer herunterladen (Link aus NinjaOne-Konsole):
wget -O ninja-agent.deb "https://app.ninjarmm.com/agent/installer/..."

# Installieren:
dpkg -i ninja-agent.deb

# Status prüfen:
systemctl status ninjarmm-agent
# Ausgabe muss "active (running)" zeigen

# Agent-Version:
/opt/NinjaRMM/programdata/ninjarmm-agent --version
```

> ✅ Der Agent registriert sich automatisch in NinjaOne – der Node erscheint innerhalb von ~2 Minuten in der Konsole.

### 1.3 Agent über Skript deployen (bei mehreren Nodes)

```bash
# deploy_ninja.sh – einmalig auf neuem Node ausführen
#!/bin/bash
INSTALLER_URL="https://app.ninjarmm.com/agent/installer/DEIN_LINK"
wget -q -O /tmp/ninja.deb "$INSTALLER_URL"
dpkg -i /tmp/ninja.deb
rm /tmp/ninja.deb
systemctl enable --now ninjarmm-agent
echo "NinjaOne Agent installiert: $(hostname)"
```

---

## 2. NinjaOne Policy für Proxmox-Nodes konfigurieren

### 2.1 Eigene Policy anlegen

```
NinjaOne → Administration → Policies → Add Policy
Name: Proxmox-Nodes
OS:   Linux
```

### 2.2 Standard-Conditions (CPU / RAM / Disk)

```
Policy → Conditions → Add

CPU-Last:
  Metric:    CPU Usage
  Operator:  greater than
  Value:     90 %
  Duration:  15 Minuten
  Severity:  Warning → Critical ab 95 %
  Notify:    E-Mail an Technik-Team

RAM-Auslastung:
  Metric:    Memory Usage
  Operator:  greater than
  Value:     85 %
  Severity:  Warning

Disk-Auslastung (System-Disk):
  Metric:    Disk Free Space
  Disk:      / (Root-Partition)
  Operator:  less than
  Value:     15 %
  Severity:  Warning → Critical unter 5 %
```

### 2.3 Service-Conditions (kritische PVE-Dienste)

```
Policy → Conditions → Service Status → Add

Dienst: pveproxy       → Status: Running  (Web-UI)
Dienst: pvedaemon      → Status: Running  (PVE-Core)
Dienst: corosync       → Status: Running  (Cluster, nur bei Cluster-Nodes)
Dienst: ninjarmm-agent → Status: Running  (Agent selbst)

Severity: Critical
Notify:   Sofort, E-Mail + ggf. Ticket in PSA
```

> ⚠️ `corosync` nur auf Cluster-Nodes überwachen – auf Single-Nodes läuft der Dienst nicht und würde Fehlalarm erzeugen.

---

## 3. Proxmox-spezifisches Monitoring via NinjaOne Script Hub

NinjaOne bietet im Script Hub offizielle Bash-Skripte für Proxmox, die Daten via `ninjarmm-cli` in Custom Fields schreiben.

### 3.1 Custom Fields anlegen

```
NinjaOne → Administration → Custom Fields → Add

Feld 1: proxmox_cluster_status   (Multiline Text)
Feld 2: proxmox_node_status      (Multiline Text)
Feld 3: proxmox_vm_info          (Multiline Text oder WYSIWYG)
Feld 4: proxmox_storage_status   (WYSIWYG)
```

### 3.2 Cluster-Status automatisch erfassen

```
NinjaOne Script Hub → suchen: "Proxmox Cluster Status"
→ Skript importieren → Automation Policy zuweisen

Parameter:
  multilineCustomField: proxmox_cluster_status
  wysiwygCustomField:   (leer lassen oder zweites WYSIWYG-Feld)

Schedule: täglich 07:00 (oder stündlich für aktives Monitoring)
```

Das Skript führt intern `pvecm status` aus und schreibt das Ergebnis formatiert in das Custom Field – sichtbar direkt im NinjaOne Device-Dashboard.

### 3.3 Node-Status erfassen

```
Script Hub → "Proxmox Node Monitoring" → importieren

# Das Skript führt aus:
pvesh get /cluster/status --noborder

# Und schreibt: Node-ID, Name, IP, Online-Status als Tabelle
# → in Custom Field: proxmox_node_status
```

### 3.4 VM-Übersicht erfassen

```
Script Hub → "Proxmox VM Information Gathering" → importieren

Parameter:
  Custom_Field_Name: proxmox_vm_info

# Listet alle VMs/Container: ID, Name, Status, RAM, CPU
# Nützlich für Kunden-Reporting und schnellen Überblick
```

### 3.5 Storage-Status erfassen

```
Script Hub → "Retrieve Proxmox Node Storage Status" → importieren

Parameter:
  multilineCustomField: proxmox_storage_status

# Zeigt: Storage-Name, Typ, Gesamtgröße, Frei, Status
# Erkennt: Datastores unter 15 % freiem Speicher
```

---

## 4. ZFS-Pool als Script Condition überwachen

NinjaOne hat keine native ZFS-Erkennung – das lösen wir mit einem Custom Script Condition.

### 4.1 Monitoring-Skript erstellen

```bash
#!/bin/bash
# zfs_health_check.sh
# NinjaOne Script Condition: Exit 0 = OK, Exit 1 = Warning/Critical

DEGRADED=$(zpool list -H -o health | grep -cv "ONLINE")

if [ "$DEGRADED" -gt 0 ]; then
    echo "ZFS ALERT: $DEGRADED Pool(s) nicht ONLINE"
    zpool status | grep -E "state:|status:|pool:"
    exit 1
fi

echo "ZFS OK: Alle Pools ONLINE"
exit 0
```

### 4.2 Skript in NinjaOne als Condition konfigurieren

```
NinjaOne → Policy → Conditions → Script → Add

Script:   zfs_health_check.sh (oben hochladen)
Schedule: alle 15 Minuten
Trigger:  Exit Code != 0
Severity: Critical
Notify:   Sofort
```

### 4.3 Cluster-Quorum überwachen

```bash
#!/bin/bash
# corosync_quorum_check.sh
# Nur auf Cluster-Nodes ausführen

if ! command -v pvecm &>/dev/null; then
    echo "Kein Cluster-Node – Skript übersprungen"
    exit 0
fi

QUORATE=$(corosync-quorumtool -p 2>/dev/null | awk '/Quorate/ {print $2}')

if [ "$QUORATE" != "Yes" ]; then
    echo "CRITICAL: Cluster hat kein Quorum! ($QUORATE)"
    corosync-quorumtool -s
    exit 1
fi

echo "OK: Cluster quorate"
exit 0
```

---

## 5. Native PVE Notifications (ergänzend)

NinjaOne-Skripte laufen zyklisch – manche Events brauchen aber sofortige Reaktion. Das PVE Notification System schickt diese direkt per E-Mail.

### 5.1 SMTP konfigurieren

```
PVE Web-UI: Datacenter → Notifications → Add → SMTP
Host:     smtp.office365.com
Port:     587, Mode: STARTTLS
Username: proxmox-alert@vireq.com
From:     proxmox@vireq.com
To:       technik@vireq.com
```

### 5.2 Notification Rules

```
Datacenter → Notifications → Notification Rules → Add

Regel 1 – Backup-Fehler (PBS):
  Matchers: severity >= error, source = backup
  Target:   SMTP

Regel 2 – S.M.A.R.T.-Fehler:
  Matchers: severity >= warning, source = system
  Target:   SMTP

Regel 3 – Cluster-Events:
  Matchers: severity >= error, source = cluster
  Target:   SMTP
```

> 💡 **Warum nicht NinjaOne dafür?** PBS-Backup-Fehler und S.M.A.R.T.-Events entstehen innerhalb von Proxmox und sind für NinjaOne ohne aufwändige Skripte nicht sofort sichtbar. Die native PVE Notification schlägt schneller an.

---

## 6. Alert-Schwellwerte Referenz

| Alarm | Schwellwert | NinjaOne-Typ | Priorität |
|-------|-------------|-------------|----------|
| CPU-Last | > 90 % für 15 Min | Condition | Warning |
| RAM-Auslastung | > 85 % | Condition | Warning |
| Disk Root-Partition | < 15 % frei | Condition | Warning |
| pveproxy gestoppt | Status != Running | Service Condition | Critical |
| pvedaemon gestoppt | Status != Running | Service Condition | Critical |
| corosync gestoppt | Status != Running | Service Condition | Critical |
| ZFS Pool degraded | Script Exit 1 | Script Condition | Critical |
| Cluster kein Quorum | Script Exit 1 | Script Condition | Critical |
| Backup-Job fehlgeschlagen | > 24h kein Backup | Native PVE Notify | Critical |
| S.M.A.R.T.-Fehler | Reallocated Sectors > 0 | Native PVE Notify | Critical |
| PBS-Datastore > 85 % | Script oder Condition | Script Condition | Warning |

---

## Best Practices

✅ NinjaOne Policy separat für Proxmox-Nodes anlegen – nicht mit Windows-Server-Policies mischen  
✅ Custom Fields für Cluster-/Node-/VM-Status täglich automatisch befüllen  
✅ ZFS- und Quorum-Checks alle 15 Minuten als Script Condition  
✅ Native PVE Notifications für Backup und S.M.A.R.T. immer aktiv halten  
✅ NinjaOne Agent in der Policy selbst überwachen (`ninjarmm-agent` Service Condition)

⚠️ Proxmox 9 (Debian 13 Trixie) läuft unter NinjaOne „Extended Support" – Agent funktioniert, aber bei Bugs kann der Support-Fix länger dauern  
⚠️ Script-Conditions laufen mit Root-Rechten – Skripte vor dem Deployment prüfen  
⚠️ `corosync`-Condition nur auf Cluster-Nodes aktiv setzen

❌ Keine separate Grafana/InfluxDB-Instanz aufbauen wenn NinjaOne bereits im Einsatz ist  
❌ Kein Monitoring-User mit Admin-Rechten in PVE – PVEAuditor-Rolle reicht für `pvesh`-Abfragen

---

## Häufige Fehler & Lösungen

**NinjaOne Agent erscheint nicht in der Konsole nach Installation**
```
Prüfen:  systemctl status ninjarmm-agent
         journalctl -u ninjarmm-agent --since "5 min ago"
Ursache: Firewall blockiert ausgehende Verbindung (Port 443)
Lösung:  Ausgehenden HTTPS-Traffic zu *.ninjarmm.com erlauben
```

**`ninjarmm-cli` nicht gefunden in Skripten**
```
Prüfen:  which ninjarmm-cli
         ls /opt/NinjaRMM/programdata/
Ursache: Agent nicht korrekt installiert oder Pfad nicht in $PATH
Lösung:  dpkg --reconfigure ninja-agent oder Neuinstallation
```

**Script Hub-Skript schlägt fehl: `pvesh command not found`**
```
Ursache: Skript läuft nicht auf Proxmox-Node oder nicht als root.
Prüfen:  which pvesh
Lösung:  In NinjaOne Policy sicherstellen: Run as = root
```

**E-Mail-Benachrichtigungen kommen nicht an**
```
Prüfen:  echo "Test" | mail -s "PVE Test" technik@vireq.com
         mailq
         journalctl -u postfix
Lösung:  /etc/postfix/main.cf prüfen, postmap /etc/postfix/sasl_passwd
```

---

## Weiterführende Links

| Ressource | URL |
|-----------|-----|
| NinjaOne Linux Agent Installation | https://www.ninjaone.com/docs/new-to-ninjaone/agent-installation/linux-device-agent-installation/ |
| NinjaOne Script Hub – Proxmox Cluster | https://www.ninjaone.com/script-hub/cluster-status-monitoring-linux/ |
| NinjaOne Script Hub – Proxmox Node | https://www.ninjaone.com/script-hub/proxmox-node-monitoring-linux/ |
| NinjaOne Script Hub – Proxmox VM Info | https://www.ninjaone.com/script-hub/proxmox-vm-information-gathering-linux/ |
| NinjaOne Script Hub – Storage Status | https://www.ninjaone.com/script-hub/retrieve-proxmox-node-storage-status/ |
| PVE Notification System | https://pve.proxmox.com/wiki/Notifications |

---

**← Vorheriges Kapitel:** [Kapitel 5 – Sicherheit & Hardening](05-sicherheit-hardening.md)  
**← Zurück zur Übersicht:** [README.md](../README.md)
