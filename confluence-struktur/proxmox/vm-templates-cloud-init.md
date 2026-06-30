<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: Proxmox -->
<!-- Title: Proxmox - VM-Templates & Cloud-Init -->
<!-- Label: pve-installation, best-practice, checkliste -->

# VM-Templates & Cloud-Init

> **PVE 9.x (Debian 13 Trixie)** · Letzte Aktualisierung: 2026-06
> Verwandte Seiten: [LVM & ZFS](lvm-zfs.md) · [Migration von Hyper-V](migration-von-hyperv.md)

---

## Wann anwenden?

- Wiederkehrende VM-Bereitstellung (Testumgebungen, Kunden-Rollouts, Entwicklungs-VMs)
- Standardisierte "Golden Images" sollen mehrfach geklont statt jedes Mal neu installiert werden
- Automatisierte Erstkonfiguration (Hostname, IP, SSH-Keys) ohne manuelles Eingreifen im Gast

---

## Überblick

Cloud-Init ist ein Industriestandard-Tool zur automatisierten Erstkonfiguration von VMs beim ersten Boot – Hostname, Benutzer, SSH-Keys, Netzwerk und Custom-Skripte werden ohne manuellen Eingriff gesetzt. Proxmox unterstützt Cloud-Init nativ über ein virtuelles CD-ROM-Laufwerk, das die Konfiguration in die VM injiziert.

```
Ohne Templates:  OS-Installation → Konfiguration → Software → ~20–30 Min. pro VM
Mit Cloud-Init:  Clone → IP/Hostname/SSH-Key setzen → Start → ~15–20 Sek. bis betriebsbereit
```

> **VIREQ-Empfehlung:** Eigene Cloud-Init-Images bauen statt fertige OpenStack-Images blind zu übernehmen – dann ist exakt bekannt, was installiert ist, und die Anpassung an Kundenanforderungen fällt leichter.

---

## Voraussetzungen

- Storage mit "Snippets"-Unterstützung, falls Custom-Cloud-Init-Configs verwendet werden sollen
- SSH-Key-Paar für passwortlosen Zugriff auf die geklonten VMs (empfohlen statt Passwort)

---

## Schritt für Schritt: Linux-Template (Debian/Ubuntu Cloud-Image)

### 1. Cloud-Image herunterladen

```bash
mkdir -p /var/lib/vz/template/iso/cloud-init/debian
cd /var/lib/vz/template/iso/cloud-init/debian

wget https://cloud.debian.org/images/cloud/trixie/latest/debian-13-genericcloud-amd64.qcow2
wget https://cloud.debian.org/images/cloud/trixie/latest/SHA512SUMS
sha512sum -c SHA512SUMS --ignore-missing
```

### 2. VM-Hülle anlegen

```bash
qm create 9000 --name debian13-cloud-template \
  --memory 2048 --cores 2 \
  --net0 virtio,bridge=vmbr0 \
  --scsihw virtio-scsi-single \
  --ostype l26 \
  --cpu cputype=x86-64-v2-AES
```

> CPU-Typ analog zur HA-Empfehlung migrationssicher wählen, wenn das Template in einem heterogenen Cluster geklont wird.

### 3. Disk-Image importieren und Cloud-Init-Laufwerk anlegen

```bash
qm importdisk 9000 debian-13-genericcloud-amd64.qcow2 local-lvm
qm set 9000 --scsi0 local-lvm:vm-9000-disk-0,discard=on
qm set 9000 --boot order=scsi0
qm set 9000 --ide2 local-lvm:cloudinit
qm set 9000 --serial0 socket --vga serial0
qm set 9000 --agent enabled=1
```

> 📸 **Screenshot machen:** VM 9000 → Hardware-Tab nach diesem Schritt – zeigt scsi0 (Disk), ide2 (CD/DVD Drive, "cloudinit"), serial0 – das ist die Referenzkonfiguration, die jede Template-VM bei euch haben soll.

### 4. Disk vergrößern

Cloud-Images liefern kleine Disks (2–3 GB) – vor dem Templating auf die gewünschte Standardgröße erweitern. Cloud-Init vergrößert das Dateisystem beim ersten Boot automatisch passend zur Disk:

```bash
qm disk resize 9000 scsi0 +18G
```

### 5. Standard-Cloud-Init-Parameter setzen

Diese Werte werden von jedem Klon geerbt, sofern nicht pro Klon überschrieben:

```bash
qm set 9000 --ciuser admin
qm set 9000 --sshkeys ~/.ssh/authorized_keys
qm set 9000 --ipconfig0 ip=dhcp
qm set 9000 --nameserver 192.168.1.1
qm set 9000 --searchdomain vireq.local
```

Auch direkt im Web UI einstellbar: **VM 9000 → Cloud-Init-Tab**

> 📸 **Screenshot machen:** VM 9000 → Cloud-Init-Tab mit ausgefülltem User, SSH-Public-Key-Feld, IP Config (DHCP), DNS-Domain – der zentrale Konfigurationsscreen, der bei jedem neuen Template wiederkehrt.

> SSH-Key-Authentifizierung wird gegenüber Passwort empfohlen – Proxmox muss bei Passwort-Auth eine verschlüsselte Version davon in den Cloud-Init-Daten speichern.

### 6. In Template umwandeln

Auch über **Web UI: Rechtsklick auf die VM → Convert to Template** möglich.

> 📸 **Screenshot machen:** Die VM-Liste im Proxmox-Web-UI mit dem geänderten Icon nach der Umwandlung (Template-Symbol statt normales VM-Symbol) – zeigt, woran man ein Template auf den ersten Blick erkennt.

```bash
qm template 9000
```

> ⚠️ **Unwiderruflich:** Ein Template kann nicht zurück in eine normale VM umgewandelt werden. Notizen zum Aufbau aufbewahren, falls das Template neu erstellt werden muss.

---

## Klonen und individuell konfigurieren

```bash
# Full Clone: unabhängige Kopie, für Produktion empfohlen
qm clone 9000 101 --name web-server-01 --full

# Linked Clone: teilt sich die Basis-Disk, schneller und platzsparender, aber abhängig vom Template
qm clone 9000 102 --name dev-server-01
```

Auch über **Web UI: Rechtsklick auf das Template → Clone** möglich.

> 📸 **Screenshot machen:** Den Clone-Dialog mit der Checkbox "Full Clone" einmal aktiviert und einmal deaktiviert (zwei Bilder oder eines mit Pfeil-Markierung) – das ist der Punkt, an dem am häufigsten versehentlich der falsche Klon-Typ gewählt wird.

```bash
# Pro Klon individuell anpassen
qm set 101 --ipconfig0 ip=192.168.1.101/24,gw=192.168.1.1
qm set 101 --name web-server-01
qm start 101
```

> **Full vs. Linked Clone:** Full Clones sind unabhängig vom Template und können auf anderes Storage verschoben werden – dafür langsamer zu erstellen und belegen vollen Speicherplatz. Linked Clones sind schneller und sparsamer, dürfen aber das Template-Storage nie verlassen und hängen am Template (das also nicht gelöscht werden darf, solange Linked Clones existieren).

---

## Custom Cloud-Init-Configs (Snippets)

Für komplexere Erstkonfiguration (Paketinstallation, Skripte) statt der automatisch generierten Konfiguration eigene YAML-Dateien verwenden:

```bash
mkdir -p /var/lib/vz/snippets
cat > /var/lib/vz/snippets/vendor.yaml << 'EOF'
#cloud-config
runcmd:
  - apt-get update
  - apt-get install -y qemu-guest-agent
  - systemctl enable --now qemu-guest-agent
EOF

qm set 9000 --cicustom "vendor=local:snippets/vendor.yaml"
```

> Snippet-Dateien müssen auf einem Storage liegen, das "Snippets" als Inhaltstyp unterstützt, und auf allen Nodes verfügbar sein, auf die die VM migriert werden könnte – sonst startet die VM dort nicht.

---

## Windows-Templates (Sysprep statt Cloud-Init)

Windows nutzt nicht Cloud-Init, sondern Sysprep in Kombination mit Cloudbase-Init für ein vergleichbares Ergebnis:

```
1. Windows-VM normal installieren, VirtIO-Treiber und QEMU Guest Agent einrichten
2. Cloudbase-Init installieren
3. Sysprep mit angepasster Unattend.xml ausführen (2 Konfigurationsdateien: 
   die normale und die *-unattend.conf – Cloudbase-Init durchläuft beide Schritte)
4. VM herunterfahren → Cloud-Init-Laufwerk hinzufügen → qm template
```

---

## Best Practices

✅ Eigene, dokumentierte Cloud-Init-Images statt blind übernommener OpenStack-Images
✅ Hohe VM-IDs (z. B. ab 9000) für Templates verwenden – sofort als solche erkennbar
✅ Naming-Konvention mit OS, Version und Erstellungsdatum (`debian13-base-20260601`)
✅ QEMU Guest Agent bereits im Template aktivieren – spart das Nachrüsten bei jedem Klon
✅ SSH-Key-Authentifizierung statt Cloud-Init-Passwort
✅ Full Clones für Produktion, Linked Clones für Test-/Dev-Umgebungen

⚠️ Template-Storage darf nicht gelöscht/verschoben werden, solange Linked Clones existieren
⚠️ `qm template` ist unwiderruflich – Notizen zum Template-Aufbau aufbewahren
⚠️ Snippet-Dateien müssen auf allen relevanten Nodes verfügbar sein

❌ Maschinenspezifische Daten (SSH-Host-Keys, machine-id) im Template belassen – führt zu Duplikaten/DHCP-Problemen bei jedem Klon
❌ Linked Clones für Produktiv-VMs mit hoher Verfügbarkeitsanforderung

---

## Häufige Fehler & Lösungen

**Geklonte VM bootet, aber Netzwerk/Hostname werden nicht gesetzt**
```
Ursache: Cloud-Init im Image nicht installiert/aktiviert, oder Datasource falsch konfiguriert.
Prüfen:  qm cloudinit dump <vmid> user; im Gast: cloud-init status; /var/log/cloud-init-output.log
```

**Mehrere Klone haben identische SSH-Host-Keys**
```
Ursache: machine-id und SSH-Host-Keys wurden vor dem Templating nicht zurücksetzt.
Lösung:  Vor qm template: echo -n > /etc/machine-id; SSH-Host-Keys löschen (regenerieren sich beim nächsten Boot via Cloud-Init).
```

**`qm clone` erstellt einen Linked statt Full Clone**
```
Ursache: --full nicht explizit angegeben.
Lösung:  qm clone <template-id> <new-id> --full 1
```

---

## Weiterführende Links

| Ressource | URL |
|---|---|
| Cloud-Init Support (offiziell) | https://pve.proxmox.com/wiki/Cloud-Init_Support |
