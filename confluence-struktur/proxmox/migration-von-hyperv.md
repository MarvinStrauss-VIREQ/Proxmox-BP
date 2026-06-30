<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: Proxmox -->
<!-- Title: Proxmox - Migration von Hyper-V -->
<!-- Label: pve-installation, best-practice, checkliste -->

# Migration von Hyper-V zu Proxmox

> **PVE 9.x (Debian 13 Trixie)** · Letzte Aktualisierung: 2026-06
> Verwandte Seiten: [VM-Templates & Cloud-Init](vm-templates-cloud-init.md) · [Single Node - Netzwerk](netzwerk-firewall-konfigurationen/single-node-netzwerk.md)

---

## Wann anwenden?

- Kunde betreibt aktuell Hyper-V und soll auf Proxmox umgestellt werden – die zentrale VIREQ-Mission
- Einzelne VMs oder ein gesamter Hyper-V-Bestand sollen übernommen werden

> **Wichtig:** Anders als bei VMware ESXi bietet Proxmox VE (Stand 9.x) **keinen integrierten Import-Wizard** für Hyper-V. Die Migration läuft manuell über Disk-Konvertierung – das ist laut offizieller Proxmox-Dokumentation der aktuelle Stand und sollte im Kundengespräch entsprechend eingeordnet werden.

---

## Überblick

Hyper-V nutzt VHD/VHDX-Diskformate, Proxmox nutzt qcow2 oder raw. Der Kern jeder Migration ist die Diskkonvertierung plus der Austausch der virtuellen Hardware (Disk-Controller, Netzwerkadapter) gegen VirtIO-Treiber, da Hyper-V-Gastsysteme diese nicht von Haus aus mitbringen.

| Methode | Eignung | Aufwand |
|---|---|---|
| Manuell: Export → `qemu-img convert` → `qm importdisk` | Volle Kontrolle, jede VM-Größe | Mittel, pro VM |
| `virt-v2v` | Batch-Konvertierung, injiziert VirtIO-Treiber automatisch | Niedrig, aber Linux-Host mit Zugriff auf beide Umgebungen nötig |
| Drittanbieter (Veeam-Restore, Vinchin, StarWind V2V) | Geeignet bei vorhandener Backup-Infrastruktur oder Zero-Downtime-Anforderung | Lizenzkosten |

> **VIREQ-Empfehlung:** Für die meisten Kundenprojekte (kein Zero-Downtime-Zwang) ist der manuelle Weg ausreichend und kostet keine zusätzliche Lizenz. `virt-v2v` lohnt sich ab einer größeren Anzahl gleichartiger VMs.

---

## Voraussetzungen

- Ziel-PVE-Node bereits eingerichtet (→ [PVE Erstinstallation](installationen/pve-erstinstallation.md), Netzwerk-Schema passend zum Anwendungsfall)
- Netzwerk-Mapping vorab dokumentiert: welcher Hyper-V-vSwitch/VLAN entspricht welcher Proxmox-Bridge/VNet
- Vollständiges Backup der Quell-VM, bevor migriert wird
- Wartungsfenster eingeplant – die Quell-VM muss für den Export herunterfahren werden (kalte Migration)

---

## Schritt für Schritt: Manuelle Migration

### 1. VirtIO-Treiber im Hyper-V-Gast vorbereiten (Windows-VMs)

Vor dem Export die VirtIO-Treiber bereits im laufenden Hyper-V-Gast installieren, damit die VM nach der Migration sofort die richtigen Treiber zur Verfügung hat:

```
Hyper-V Manager → VM → Settings → DVD Drive → Image file: virtio-win-x.x.x.iso
Im Gast: virtio-win-gt-x64.msi ausführen (installiert Storage-, Netzwerk- und Balloon-Treiber)
QEMU Guest Agent ebenfalls aus dem ISO installieren (Verzeichnis guest-agent)
```

> Linux-Gastsysteme bringen VirtIO-Treiber bereits im Kernel mit – dieser Schritt entfällt dort.

### 2. VM in Hyper-V exportieren

```
Hyper-V Manager → VM Rechtsklick → Export
Die VHDX-Datei liegt anschließend im Export-Verzeichnis unter "Virtual Hard Disks"
```

### 3. Disk auf den Proxmox-Host übertragen

```
WinSCP: Verbindung zum Proxmox-Host aufbauen, VHDX per Drag & Drop übertragen
```

```bash
# Integrität der übertragenen Datei prüfen
qemu-img check -r all /root/import/MeineVM.vhdx
```

### 4. VM-Hülle in Proxmox anlegen

```bash
qm create 300 --name MeineVM --memory 4096 --cores 4 \
  --net0 virtio,bridge=vmbr1 \
  --ostype l26   # bei Windows-Gast: --ostype win11 / win10 / win2k22 etc.
```

### 5. Disk konvertieren und einbinden

```bash
qemu-img convert -O qcow2 /root/import/MeineVM.vhdx /var/lib/vz/images/300/vm-300-disk-0.qcow2

qm rescan
qm set 300 --scsi0 local-lvm:vm-300-disk-0 --scsihw virtio-scsi-single
qm set 300 --boot order=scsi0
```

### 6. Erster Boot mit generischen Treibern (Windows-VMs)

Windows-Gastsysteme, die nie VirtIO genutzt haben, booten beim ersten Versuch mit `virtio-scsi` nicht (`INACCESSIBLE_BOOT_DEVICE`). Workaround:

```bash
# Disk-Controller temporär auf SATA setzen, damit Windows mit Inbox-Treibern bootet
qm set 300 --sata0 local-lvm:vm-300-disk-0
qm set 300 --boot order=sata0
qm set 300 --ide2 local:iso/virtio-win.iso,media=cdrom
```

Nach erfolgreichem Boot: VirtIO-Storage-Treiber (`vioscsi`) aus dem gemounteten ISO installieren, dann VM herunterfahren, Disk-Controller auf `virtio-scsi-single` umstellen und neu starten.

### 7. BIOS/UEFI-Modus prüfen

```
Gen1-VM (Hyper-V) → Proxmox: BIOS = SeaBIOS
Gen2-VM (Hyper-V, UEFI)     → Proxmox: BIOS = OVMF (UEFI) + EFI-Disk im Hardware-Panel anlegen
```

> Manche Betriebssysteme setzen im UEFI-Modus keinen Standard-Bootpfad (`/EFI/BOOT/BOOTX64.EFI`), sondern einen eigenen. Falls die VM trotz korrektem BIOS nicht startet, manuell in die UEFI-Shell wechseln und den Boot-Eintrag ergänzen.

### 8. Netzwerk im Gast nachkonfigurieren

```bash
# Linux:
ip addr show   # Interface-Namen können sich ändern (ens18 statt eth0)

# Windows: VirtIO-Netzwerktreiber (NetKVM) installieren, dann statische IP/DNS
# laut vorher dokumentierter Konfiguration neu eintragen
```

### 9. Alte Hypervisor-Tools entfernen

Vor der Migration im Gast-OS die Hyper-V-Integrationsdienste/Hyper-V-spezifischen Treiber deinstallieren – nachträglich ist das oft schwieriger als vorher.

---

## Alternative: virt-v2v (Batch-Migration)

Für mehrere VMs oder wenn die Treiber-Injektion automatisiert werden soll:

```bash
# Von einem Linux-Host mit Zugriff auf die VHDX-Datei:
virt-v2v -i disk /pfad/zur/MeineVM.vhdx -o local -os /var/tmp

# Ergebnis nach Proxmox importieren
qm importdisk 300 /var/tmp/MeineVM-sda.qcow2 local-lvm
```

`virt-v2v` konvertiert die Disk, passt die Gast-Konfiguration an und injiziert VirtIO-Treiber automatisch – das spart insbesondere bei Windows-VMs Schritt 6.

---

## Funktionstest nach der Migration

- **Boot-Test:** VM bootet ohne Fehler, Konsole zeigt erwartetes OS
- **Netzwerk:** IP-Adressierung, DNS-Auflösung, Konnektivität zu abhängigen Diensten
- **Anwendungstest:** alle Dienste starten und funktionieren korrekt
- **Performance-Baseline:** Disk-I/O, Netzwerk-Durchsatz, CPU-Auslastung gegen Vorher-Werte vergleichen
- **Backup-Verifizierung:** VM in die PBS-Backup-Jobs aufnehmen, Testbackup durchführen

```bash
apt install qemu-guest-agent   # Linux
systemctl enable --now qemu-guest-agent
qm agent 300 ping              # vom Host aus prüfen
```

---

## Best Practices

✅ VirtIO-Treiber **vor** dem Export im laufenden Hyper-V-Gast installieren – spart einen Boot-Zyklus mit SATA-Workaround
✅ Netzwerk-Mapping (vSwitch → Bridge/VNet) vor der Migration schriftlich dokumentieren
✅ Erste 3–5 VMs auf einem separaten Test-Node migrieren, bevor produktive VMs angefasst werden
✅ Vollständiges Backup vor jeder Migration – die Quell-VM bleibt bis zur Verifizierung unangetastet

⚠️ Snapshots in der Quell-Hyper-V-Umgebung vorher konsolidieren – angehäufte Snapshots verlangsamen den Export unvorhersehbar
⚠️ Anwendungen, die an MAC-Adresse, Disk-Seriennummer oder Hardware-ID gebunden sind, können nach der Migration nicht mehr starten – vorher prüfen

❌ Hyper-V-Integrationsdienste im Gast belassen und erst nachträglich entfernen
❌ Migration ohne vorheriges Netzwerk-Mapping durchführen – VM bootet, landet aber im falschen Segment

---

## Häufige Fehler & Lösungen

**Windows bootet mit `INACCESSIBLE_BOOT_DEVICE`**
```
Ursache: virtio-scsi ohne installierten Treiber als Bootdisk gesetzt.
Lösung:  Schritt 6 – temporär auf SATA booten, VirtIO-Treiber installieren, dann umstellen.
```

**VM bootet, aber kein UEFI-Bootmenü / kein Bootpfad gefunden**
```
Ursache: Custom-Bootpfad statt Standard-EFI-Pfad, oder EFI-Disk fehlt.
Lösung:  EFI-Disk im Hardware-Panel anlegen; in der UEFI-Shell Boot-Eintrag ergänzen.
```

**Disk-Kette bricht ab – Basis-Image fehlt nach Kopie**
```
Ursache: Nur die Differenz-/Overlay-Disk kopiert, Basis-Image (Snapshot-Kette) blieb zurück.
Lösung:  Disk-Kette vor dem Export auf der Quelle flatten/konsolidieren.
```

---

## Weiterführende Links

| Ressource | URL |
|---|---|
| Migrate to Proxmox VE (offiziell) | https://pve.proxmox.com/wiki/Migrate_to_Proxmox_VE |
| Advanced Migration Techniques | https://pve.proxmox.com/wiki/Advanced_Migration_Techniques_to_Proxmox_VE |
| VirtIO-Win Treiber-ISO | https://github.com/virtio-win/virtio-win-pkg-scripts |
