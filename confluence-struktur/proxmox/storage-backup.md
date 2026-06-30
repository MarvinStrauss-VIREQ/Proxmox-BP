<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: Proxmox -->
<!-- Title: Proxmox - Storage & Backup -->
<!-- Label: pve-storage, best-practice, checkliste -->

# Storage & Backup

> **PVE 9.x / PBS 4.x** · Letzte Aktualisierung: 2026-06
> Quelle: adaptiert aus Kapitel 03 – Backup-Strategien. **Vor Sync gegen bestehenden Confluence-Stand prüfen.**
> Verwandte Seiten: [Backup Server Erstinstallation](installationen/backup-server-erstinstallation.md) · [backup Server - Netzwerk](netzwerk-firewall-konfigurationen/backup-server-netzwerk.md) · [LVM & ZFS](lvm-zfs.md)

---

## Wann anwenden?

- Backup-Datastore auf PBS anlegen und in PVE einbinden
- Backup-Jobs, Retention-Policy und Verifizierung konfigurieren
- Garbage Collection planen und überwachen
- VM oder einzelne Dateien wiederherstellen

> Installation des PBS-Servers selbst → [Backup Server Erstinstallation](installationen/backup-server-erstinstallation.md). Netzwerk-Layout → [backup Server - Netzwerk](netzwerk-firewall-konfigurationen/backup-server-netzwerk.md).

---

## Überblick

| Methode | Beschreibung | Empfehlung |
|---|---|---|
| `vzdump` | Eingebaut, schreibt tar/vma direkt ins Dateisystem | Einfach, lokale Backups |
| Proxmox Backup Server (PBS) | Dedizierter Server, Dedup, Verschlüsselung, Verifizierung | ✅ Produktion |

**PBS-Vorteile:** inkrementelle Backups (nur geänderte Chunks), Chunk-basierte Deduplizierung (bis 80 % Platzersparnis), AES-256-Verschlüsselung, automatische Verifizierung, granulare Wiederherstellung einzelner Dateien.

> **Hyper-V-Vergleich:** PBS ≈ Hornetsecurity/Altaro VM Backup auf NAS – mit nativem Block-Level-Dedup und integrierter Verifizierung.

---

## Backup-Datastore anlegen

```bash
# ZFS-Pool für Backups (auf PBS):
zpool create -o ashift=12 backuppool mirror \
  /dev/disk/by-id/DISK1 \
  /dev/disk/by-id/DISK2
zfs set compression=lz4 backuppool

proxmox-backup-manager datastore create main-backup /backuppool
```

## PBS in PVE einbinden

```bash
# Fingerprint ermitteln (auf PBS):
proxmox-backup-manager cert info | grep Fingerprint

# Als Storage in PVE registrieren:
pvesm add pbs pbs-backup \
    --server 192.168.1.50 \
    --datastore main-backup \
    --username backup-user@pbs \
    --password PASSWORT \
    --content backup \
    --fingerprint XX:XX:XX:...
```

---

## Backup-Jobs konfigurieren

```
Datacenter → Backup → Add
Storage:        pbs-backup
Schedule:       täglich 02:00 (Cron: "0 2 * * *")
Selection mode: All / Include / Exclude
Mode:           Snapshot (kein Downtime, empfohlen) | Suspend | Stop (maximale Konsistenz)
Compression:    zstd
```

### Retention-Policy

```bash
proxmox-backup-manager datastore update main-backup \
    --keep-last 3 \
    --keep-daily 7 \
    --keep-weekly 4 \
    --keep-monthly 6
```

| Parameter | Wert | Bedeutung |
|---|---|---|
| `keep-last` | 3 | Letzte 3 Backups immer |
| `keep-daily` | 7 | 1 Backup/Tag für 7 Tage |
| `keep-weekly` | 4 | 1 Backup/Woche für 4 Wochen |
| `keep-monthly` | 6 | 1 Backup/Monat für 6 Monate |

### Verifizierung

```bash
# PBS Web-UI: Datastore → main-backup → Verify Jobs → Add (wöchentlich empfohlen)
proxmox-backup-client snapshot verify STORE TYPE/ID/DATUM
```

---

## Garbage Collection

> Bereinigt verwaiste Daten-Chunks nach Ablauf von Backups – ohne GC wächst der Datastore unbegrenzt.

```bash
proxmox-backup-manager garbage-collection start main-backup
proxmox-backup-manager garbage-collection status main-backup
```

---

## Wiederherstellung

**Komplette VM:** `PVE Web-UI → Storage → pbs-backup → Backups → Restore` (Ziel-Node, neue VM-ID, Ziel-Storage wählen).

**Einzelne Dateien (granular):**

```bash
proxmox-backup-client mount \
    pbs-backup:backup/vm/100/2026-06-01T02:00:00Z \
    /mnt/restore

cp /mnt/restore/etc/wichtig.conf /tmp/
proxmox-backup-client umount /mnt/restore
```

---

## Best Practices

✅ Verifizierung wöchentlich automatisch – nur verifizierte Backups sind verlässlich
✅ GC täglich planen
✅ 3-2-1-Regel: 3 Kopien, 2 Medien, 1 offsite
✅ Enterprise-SSDs mit PLP für den Backup-ZFS-Pool

⚠️ Backup-Mode `Stop` ist am konsistentesten, erzeugt aber Downtime
⚠️ Retention-Policy greift erst beim nächsten Prune-Lauf
⚠️ Dedup-Index wächst mit der Zeit – RAM-Formel regelmäßig neu prüfen (→ Backup Server Erstinstallation)

❌ Backup- und VM-Storage auf denselben physischen Disks
❌ Consumer-SSDs ohne PLP für den PBS-ZFS-Pool

---

## Häufige Fehler & Lösungen

**`unable to open datastore 'main-backup'`**
```
Ursache: Falscher Fingerprint oder falsche Zugangsdaten.
Lösung:  pvesm set pbs-backup --fingerprint NEUER_FINGERPRINT
```

**Backup schlägt fehl: VM is locked**
```
Lösung: qm unlock VM-ID
```

**PBS-Speicher wächst trotz Retention**
```
Ursache: Garbage Collection nicht konfiguriert.
Lösung:  proxmox-backup-manager garbage-collection start main-backup
```

---

## Weiterführende Links

| Ressource | URL |
|---|---|
| PBS Admin Guide | https://pbs.proxmox.com/docs/pbs-admin-guide.html |
| Backup & Restore | https://pve.proxmox.com/wiki/Backup_and_Restore |
