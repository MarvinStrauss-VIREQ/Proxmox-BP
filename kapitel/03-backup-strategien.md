# Kapitel 3: Backup-Strategien

> **Version:** Proxmox VE 9.x / PBS 4.x · **Letzte Aktualisierung:** 2026-06  
> **Verwandte Kapitel:** [Kapitel 2 – Netzwerk & Clustering](02-netzwerk-clustering.md) · [Kapitel 5 – Sicherheit](05-sicherheit-hardening.md)

---

## Wann anwenden?

> - Proxmox Backup Server (PBS) installieren und konfigurieren
> - Backup-Jobs für VMs und Container einrichten
> - PBS zusätzlich als QDevice für 2-Node-Cluster nutzen
> - Retention, Verifizierung und Wiederherstellung konfigurieren

---

## Überblick

| Methode | Beschreibung | Empfehlung |
|---------|-------------|----------|
| **vzdump** | Eingebaut, schreibt tar/vma direkt ins Dateisystem | Einfach, lokale Backups |
| **Proxmox Backup Server (PBS)** | Dedizierter Server, Dedup, Verschlüsselung, Verifizierung | ✅ Produktion |

**PBS-Vorteile:**
- Inkrementelle Backups (nur geänderte Chunks)
- Chunk-basierte Deduplizierung (bis 80% Platzersparnis)
- AES-256-Verschlüsselung
- Automatische Verifizierung
- Granulare Wiederherstellung (einzelne Dateien)
- **QDevice-Doppelrolle** für 2-Node-Cluster

> **Hyper-V-Vergleich:** PBS ≈ Hornetsecurity/Altaro VM Backup auf NAS – mit nativem Block-Level-Dedup, integrierter Verifizierung und QDevice-Funktion.

---

## Voraussetzungen PBS-Server

| Komponente | Minimum | Produktion |
|-----------|---------|----------|
| CPU | 2 Kerne | 4+ Kerne |
| RAM | 4 GB | 8–16 GB |
| System-Disk | 32 GB SSD | 64 GB SSD |
| Backup-Storage | Beliebig | **Enterprise SSD mit PLP** |
| NIC | 1 × 1G | 1 × 10G |

> ❌ **Consumer-SSDs ohne PLP (Power Loss Protection)** können bei Stromausfall den ZFS-Pool korrupieren. Auf PBS sind Enterprise-SSDs Pflicht.

**RAM-Formel PBS:**
```
RAM = 4 GB (OS) + 1 GB × [Dedup-Index in TB] + 1 GB × [ZFS-Pool in TB]
```
Beispiel 10 TB Pool: 4 + 10 + 10 = **24 GB RAM**

---

## 1. PBS Installation

```bash
# ISO: https://www.proxmox.com/de/downloads/proxmox-backup-server/iso
# Installation identisch zu PVE (Kapitel 1, Abschnitt 2.3)
# Empfehlung: ZFS Mirror für System-Disk
# Web-UI nach Install: https://PBS-IP:8007

# Post-Install Repository (PBS 4.x / Debian Trixie):
cat > /etc/apt/sources.list.d/pbs-enterprise.list << 'EOF'
# deb https://enterprise.proxmox.com/debian/pbs trixie pbs-enterprise
EOF

cat >> /etc/apt/sources.list << 'EOF'
deb http://download.proxmox.com/debian/pbs trixie pbs-no-subscription
EOF

apt update && apt full-upgrade -y
```

> **PBS 3.x (Debian Bookworm):** `trixie` → `bookworm`

---

## 2. Backup-Datastore anlegen

```bash
# ZFS-Pool für Backups (auf PBS):
zpool create -o ashift=12 backuppool mirror \
  /dev/disk/by-id/DISK1 \
  /dev/disk/by-id/DISK2

zfs set compression=lz4 backuppool

# Datastore anlegen:
# Web-UI: Administration → Datastore → Create
# Name: main-backup, Path: /backuppool

# Via CLI:
proxmox-backup-manager datastore create main-backup /backuppool
```

---

## 3. PBS in PVE einbinden

```bash
# Fingerprint des PBS ermitteln (auf PBS):
proxmox-backup-manager cert info | grep Fingerprint
# Oder Web-UI: Administration → Certificates

# PBS als Storage in PVE registrieren:
pvesm add pbs pbs-backup \
    --server 192.168.1.50 \
    --datastore main-backup \
    --username backup-user@pbs \
    --password PASSWORT \
    --content backup \
    --fingerprint XX:XX:XX:...

# Web-UI: Datacenter → Storage → Add → Proxmox Backup Server
```

---

## 4. PBS als QDevice (2-Node-Cluster)

> PBS dient gleichzeitig als Backup-Server **und** Quorum-Witness. Details → [Kapitel 2, Abschnitt 4](02-netzwerk-clustering.md).

```bash
# Auf PBS:
apt install -y corosync-qnetd
systemctl enable --now corosync-qnetd

# Auf pve01:
pvecm qdevice setup 192.168.1.50
pvecm status  # Muss 3 Votes zeigen
```

---

## 5. Backup-Jobs konfigurieren

### 5.1 Backup-Job anlegen (Web-UI)

```
Datacenter → Backup → Add
Storage:        pbs-backup
Schedule:       täglich 02:00 (Cron: "0 2 * * *")
Selection mode: All (alle VMs) / Include / Exclude
Mode:           Snapshot  (kein Downtime, empfohlen)
              │ Suspend   (kurze Pause, konsistenter)
              └ Stop      (VM gestoppt, maximale Konsistenz)
Compression:    zstd
Notes template: {{guestname}} - {{node}} - {{date}}
```

### 5.2 Retention-Policy

```bash
# PBS Web-UI: Datastore → main-backup → Options → Retention
# Empfehlung Produktion:

proxmox-backup-manager datastore update main-backup \
    --keep-last 3 \
    --keep-daily 7 \
    --keep-weekly 4 \
    --keep-monthly 6
```

| Parameter | Wert | Bedeutung |
|-----------|------|-----------|
| keep-last | 3 | Letzte 3 Backups immer |
| keep-daily | 7 | 1 Backup/Tag für 7 Tage |
| keep-weekly | 4 | 1 Backup/Woche für 4 Wochen |
| keep-monthly | 6 | 1 Backup/Monat für 6 Monate |

### 5.3 Verifizierung aktivieren

```bash
# PBS Web-UI: Datastore → main-backup → Verify Jobs → Add
# Schedule: wöchentlich (prüft Checksummen und Lesbarkeit)

# Manuelle Verifizierung:
proxmox-backup-client snapshot verify STORE TYPE/ID/DATUM
```

---

## 6. Garbage Collection

> **GC** bereinigt verwaiste Daten-Chunks nach Ablauf von Backups. Ohne GC wächst der Datastore unbegrenzt.

```bash
# Automatisch (PBS Web-UI): Datastore → GC Jobs → täglich

# Manuell:
proxmox-backup-manager garbage-collection start main-backup
proxmox-backup-manager garbage-collection status main-backup
```

---

## 7. Wiederherstellung

### 7.1 Komplette VM wiederherstellen

```
PVE Web-UI: Storage → pbs-backup → Backups
→ Gewünschtes Backup wählen → Restore
Ziel-Node, neue VM-ID, Ziel-Storage auswählen
Optional: „Start after restore“ aktivieren
```

### 7.2 Einzelne Dateien wiederherstellen (granular)

```
PBS Web-UI: Datastore → main-backup → Backup auswählen → File Browser
→ Einzelne Dateien herunterladen oder in VM zurückkopieren
```

```bash
# Via CLI (proxmox-backup-client):
proxmox-backup-client mount \
    pbs-backup:backup/vm/100/2026-06-01T02:00:00Z \
    /mnt/restore

cp /mnt/restore/etc/wichtig.conf /tmp/
proxmox-backup-client umount /mnt/restore
```

---

## Best Practices

✅ PBS auf dedizierter Hardware (nicht als VM auf den gesicherten Nodes)  
✅ Enterprise-SSDs mit PLP für ZFS-Backup-Pool  
✅ PBS in separatem Backup-VLAN  
✅ Verifizierung wöchentlich automatisch — nur verifizierte Backups sind verlässlich  
✅ GC täglich planen  
✅ 3-2-1-Regel: 3 Kopien, 2 Medien, 1 offsite

⚠️ Backup-Mode `Stop` ist konsistentester, erzeugt aber Downtime  
⚠️ Retention-Policy greift erst beim nächsten Prune-Lauf  
⚠️ Dedup-Index wächst mit der Zeit – RAM-Formel regelmäßig neu prüfen

❌ PBS als VM auf dem gesicherten Cluster betreiben  
❌ Backup- und VM-Storage auf denselben physischen Disks  
❌ Consumer-SSDs (ohne PLP) für PBS-ZFS-Pool

---

## Häufige Fehler & Lösungen

**`unable to open datastore 'main-backup'`**
```
Ursache: Falscher Fingerprint oder falsche Zugangsdaten.
Lösung:  pvesm set pbs-backup --fingerprint NEUER_FINGERPRINT
Prüfen:  openssl s_client -connect PBS-IP:8007 | openssl x509 -fingerprint
```

**Backup schlägt fehl: VM is locked**
```
Ursache: VM durch anderen Task gesperrt.
Lösung:  qm unlock VM-ID
```

**PBS-Speicher wächst trotz Retention**
```
Ursache: Garbage Collection nicht konfiguriert.
Lösung:  proxmox-backup-manager garbage-collection start main-backup
```

---

## Weiterführende Links

| Ressource | URL |
|-----------|-----|
| PBS Admin Guide | https://pbs.proxmox.com/docs/pbs-admin-guide.html |
| Backup & Restore | https://pve.proxmox.com/wiki/Backup_and_Restore |
| PBS als QDevice | https://pve.proxmox.com/wiki/Cluster_Manager#_corosync_external_vote_support |

---

**→ Nächstes Kapitel:** [Kapitel 4 – Datacenter Manager](04-datacenter-manager.md)  
**← Vorheriges Kapitel:** [Kapitel 2 – Netzwerk & Clustering](02-netzwerk-clustering.md)
