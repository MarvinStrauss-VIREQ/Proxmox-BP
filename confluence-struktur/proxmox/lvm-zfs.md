<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: Proxmox -->
<!-- Title: Proxmox - LVM & ZFS -->
<!-- Label: pve-storage, best-practice, referenz -->

# LVM & ZFS

> **PVE 9.x (Debian 13 Trixie)** · Letzte Aktualisierung: 2026-06
> Quelle: ergänzt aus Kapitel 01, Abschnitt 2.4 und 5. **Vor Sync gegen bestehenden Confluence-Stand prüfen.**
> Verwandte Seiten: [PVE Erstinstallation](installationen/pve-erstinstallation.md) · [Storage & Backup](storage-backup.md)

---

## Wann anwenden?

- Bei der Entscheidung, welches Dateisystem für OS-Disk und VM-Storage eingesetzt wird
- Beim Anlegen eines separaten Storage-Pools für VM-Disks
- Beim Erweitern von Storage (zusätzliche Disks, Thin-Pool-Erweiterung)

---

## Entscheidungstabelle: ext4 / ZFS / LVM

| Szenario | Empfehlung | Begründung |
|---|---|---|
| Einzelne OS-Disk | **ext4** | ZFS auf einer Disk bietet keine Redundanz, belegt aber RAM für ARC – ext4 ist effizienter (VIREQ-Standard) |
| Zwei gleichartige Disks | **ZFS Mirror** | Redundanz für OS und VM-Storage in einem, Self-Healing, schnelle Snapshots |
| Drei oder mehr Disks | **ZFS RAIDZ-1/2** | RAIDZ-1 ab 3 Disks, RAIDZ-2 ab 4 Disks |
| VM-Storage bei ext4-OS | **LVM-Thin** (`local-lvm`, Standard) oder separater ZFS-Pool | LVM-Thin ist der Proxmox-Default ohne Zusatzaufwand; ZFS-Pool wenn Snapshots/Self-Healing gewollt sind |

> **VIREQ-Standard:** ext4 für die OS-Disk, kein ZFS auf Einzeldisks. Bei nur einer NVMe-OS-Disk und separaten Storage-Disks: dedizierter ZFS-Pool für VM-Disks (→ Abschnitt "ZFS-Pool anlegen").

---

## LVM – Proxmox-Standard-Storage

Nach einer ext4-Installation legt Proxmox automatisch zwei Storages an:

| Storage | Typ | Inhalt |
|---|---|---|
| `local` | Directory `/var/lib/vz` | ISOs, Templates, Backups |
| `local-lvm` | LVM-Thin | VM-Disks, Container |

### LVM-Grundbefehle

```bash
# Physical Volumes, Volume Groups, Logical Volumes anzeigen
pvs
vgs
lvs

# Thin-Pool-Auslastung prüfen (kritisch – Thin-Pools können "overcommitted" sein)
lvs -a -o +data_percent,metadata_percent
```

### Zusätzliche Disk zu einer Volume Group hinzufügen

```bash
pvcreate /dev/disk/by-id/NEUE-DISK
vgextend pve /dev/disk/by-id/NEUE-DISK

# Thin-Pool erweitern (Beispiel: data-Pool in VG "pve")
lvextend -l +100%FREE pve/data
```

> ⚠️ **Thin-Pool-Overcommit:** LVM-Thin erlaubt, mehr virtuelle Kapazität zuzuweisen als physisch vorhanden ist. `data_percent` regelmäßig überwachen (→ NinjaOne-Monitoring) – ein voller Thin-Pool kann laufende VMs in einen Read-Only-Zustand zwingen.

---

## ZFS – separater Pool für VM-Storage

> Immer `/dev/disk/by-id/` verwenden – nie `/dev/sdX`, da sich Gerätenamen nach einem Reboot ändern können.

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,TYPE
ls -l /dev/disk/by-id/ | grep -v part

# ZFS Mirror aus zwei Storage-Disks:
zpool create -o ashift=12 vmpool mirror \
  /dev/disk/by-id/DISK1 \
  /dev/disk/by-id/DISK2

zfs set compression=lz4 vmpool
zpool status vmpool

# In PVE registrieren:
pvesm add zfspool vmpool --pool vmpool --content images,rootdir
```

> **`ashift=12`** ist für moderne SSDs/NVMe korrekt und lässt sich **nachträglich nicht ändern** – vor dem Anlegen prüfen.

### ZFS ARC begrenzen

Nur relevant, wenn ZFS tatsächlich genutzt wird (OS oder separater Pool):

**Formel:** `RAM = 2 GB OS + 1 GB/TB ZFS-Pool + VM-RAM`

```bash
# Aktuellen ARC prüfen:
awk '/^size/{printf "ARC: %.1f GB\n", $3/1073741824}' /proc/spl/kstat/zfs/arcstats

# Auf 4 GB begrenzen (Beispiel):
cat > /etc/modprobe.d/zfs.conf << 'EOF'
options zfs zfs_arc_max=4294967296
options zfs zfs_arc_min=1073741824
EOF
update-initramfs -u
```

### ZFS Snapshots

```bash
zfs snapshot vmpool@vor-update
zfs list -t snapshot
zfs rollback vmpool@vor-update
```

### ZFS Scrub – regelmäßige Integritätsprüfung

```bash
# Manuell:
zpool scrub vmpool
zpool status vmpool   # Scrub-Fortschritt und Fehler

# Proxmox legt standardmäßig einen monatlichen Cron-Job an – prüfen:
cat /etc/cron.d/zfsutils-linux
```

---

## NFS-Share einbinden

```bash
apt install -y nfs-common
showmount -e 192.168.1.100

pvesm add nfs nas-backup \
  --server 192.168.1.100 \
  --export /proxmox-share \
  --content backup,iso \
  --options vers=4.1
```

---

## Best Practices

✅ ext4 für Einzeldisk-OS, ZFS Mirror/RAIDZ ab zwei Disks (VIREQ-Standard)
✅ VM-Storage immer auf separatem ZFS-Pool oder LVM-Thin, nie auf der OS-Disk-Partition direkt erweitern
✅ `ashift=12` bei modernen SSD/NVMe – vor dem Anlegen prüfen, da nicht nachträglich änderbar
✅ Thin-Pool-Auslastung (`lvs -a -o +data_percent`) ins Monitoring aufnehmen
✅ ZFS-Scrub-Ergebnisse regelmäßig prüfen

⚠️ Hardware-RAID-Controller sind mit ZFS inkompatibel – HBA-IT-Mode erforderlich (→ Kapitel 01, Abschnitt 1.2)
⚠️ ZFS ARC kann ohne Limit VM-RAM verdrängen

❌ `/dev/sdX` statt `/dev/disk/by-id/` beim Pool-Anlegen verwenden
❌ Thin-Pool ohne Monitoring der Auslastung betreiben

---

## Weiterführende Links

| Ressource | URL |
|---|---|
| OpenZFS Docs | https://openzfs.github.io/openzfs-docs/ |
| PVE Storage – LVM | https://pve.proxmox.com/pve-docs/pve-admin-guide.html#chapter_lvm |
| PVE Storage – ZFS | https://pve.proxmox.com/pve-docs/pve-admin-guide.html#chapter_zfs |
