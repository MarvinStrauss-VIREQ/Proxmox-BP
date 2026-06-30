<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: Proxmox -->
<!-- Title: Proxmox - PCI/GPU-Passthrough -->
<!-- Label: pve-installation, best-practice, checkliste -->

# PCI/GPU-Passthrough

> **PVE 9.x (Debian 13 Trixie)** · Letzte Aktualisierung: 2026-06
> Voraussetzung: IOMMU im Kernel aktiviert (→ [PVE Erstinstallation](installationen/pve-erstinstallation.md) / Kapitel 01, Abschnitt 4.9)

---

## Wann anwenden?

- GPU-Passthrough für CAD/Rendering-VMs, KI/ML-Workloads (CUDA/ROCm) oder VDI-Gaming-Anwendungsfälle
- PCI-Passthrough für High-Performance-Netzwerkkarten, RAID-Controller oder Capture-Cards in einer VM
- Wenn echte Hardware-Performance benötigt wird, die virtualisierte/emulierte Geräte nicht liefern können

> Passthrough bindet die Hardware exklusiv an eine VM – der Proxmox-Host selbst kann das Gerät danach nicht mehr nutzen (Ausnahme: SR-IOV, siehe unten).

---

## Überblick

PCI(e)-Passthrough nutzt VFIO (Virtual Function I/O), ein Linux-Kernel-Framework, das ein Hardware-Gerät sicher direkt einer VM zuweist – die VM sieht das Gerät mit nativen Treibern, fast ohne Virtualisierungs-Overhead.

**Voraussetzungen an die Hardware:**

| Anforderung | Prüfung |
|---|---|
| CPU mit IOMMU-Unterstützung | Intel VT-d oder AMD-Vi – die meisten CPUs der letzten 5–6 Jahre |
| Mainboard mit IOMMU-Option im BIOS | Manche Consumer-Boards verstecken die Option unter anderem Namen |
| Saubere IOMMU-Gruppe | Das Gerät sollte in einer eigenen IOMMU-Gruppe liegen, sonst muss die gesamte Gruppe durchgereicht werden |
| Zweite GPU für den Host empfohlen | Eine Basis-GPU/iGPU für die Proxmox-Konsole, die durchgereichte GPU exklusiv für die VM |

---

## Schritt 1: IOMMU aktivieren

Bereits in der Grundkonfiguration beschrieben (→ [PVE Erstinstallation](installationen/pve-erstinstallation.md)) – hier nochmal als Voraussetzungscheck:

```bash
dmesg | grep -e DMAR -e IOMMU
# Erwartet: "DMAR: IOMMU enabled" (Intel) oder "AMD-Vi: IOMMU enabled" (AMD)
```

> Bei Installation auf ZFS wird `/etc/default/grub` **nicht** verwendet – Kernel-Parameter stattdessen in `/etc/kernel/cmdline` setzen und mit `proxmox-boot-tool refresh` aktivieren:

```bash
# Nur bei ZFS-Root-Installation:
nano /etc/kernel/cmdline
# z.B.: root=ZFS=rpool/ROOT/pve-1 boot=zfs quiet intel_iommu=on iommu=pt
proxmox-boot-tool refresh
```

## Schritt 2: IOMMU-Gruppen prüfen

```bash
for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do
  echo "IOMMU Group $(basename $g):"
  for d in $g/devices/*; do
    echo -e "\t$(lspci -nns $(basename $d))"
  done
done
```

> Liegen mehrere unabhängige Geräte in derselben Gruppe wie die GPU, müssen alle zusammen durchgereicht werden – oder die GPU in einen anderen PCIe-Slot stecken, der eine saubere Gruppierung ermöglicht. Ein ACS-Override-Patch (`pcie_acs_override=downstream,multifunction`) kann als letztes Mittel helfen, hat aber Isolations-Nachteile und sollte bewusst gewählt werden.

## Schritt 3: GPU isolieren (VFIO statt Host-Treiber)

```bash
# Vendor:Device-ID ermitteln
lspci -nn | grep -i nvidia   # oder amd/vga je nach Hersteller
# Beispiel-Ausgabe: 01:00.0 VGA ... [10de:2503]

# VFIO-Bindung erzwingen
echo "options vfio-pci ids=10de:2503,10de:228e disable_vga=1" > /etc/modprobe.d/vfio.conf

# Host-Treiber blacklisten, damit sie das Gerät nicht vorher greifen
cat > /etc/modprobe.d/blacklist-gpu.conf << 'EOF'
blacklist nouveau
blacklist nvidia
blacklist nvidiafb
blacklist nvidia_drm
blacklist radeon
blacklist amdgpu
EOF

# VFIO-Module laden
echo "vfio
vfio_iommu_type1
vfio_pci" >> /etc/modules

update-initramfs -u -k all
reboot
```

## Schritt 4: Bindung verifizieren

```bash
lspci -nnk -d 10de:   # Vendor-ID anpassen
# Kernel driver in use: vfio-pci   → korrekt
# Steht dort noch nouveau/nvidia/amdgpu/radeon → Blacklist hat nicht gegriffen
```

## Schritt 5: VM für Passthrough konfigurieren

Für beste Kompatibilität: Maschinentyp `q35`, BIOS `OVMF` (UEFI) statt SeaBIOS, PCIe statt PCI.

```bash
qm set <vmid> --machine q35
qm set <vmid> --bios ovmf
qm set <vmid> --efidisk0 local-lvm:1,efitype=4m,pre-enrolled-keys=1
qm set <vmid> --cpu host

# GPU als Hardware hinzufügen (Web UI: Hardware → Add → PCI Device, oder per CLI):
qm set <vmid> --hostpci0 01:00,pcie=1,x-vga=1
```

| Option | Bedeutung |
|---|---|
| `x-vga=1` | markiert das Gerät als primäre GPU der VM |
| `pcie=1` | nutzt einen PCIe- statt legacy-PCI-Port (nötig für `q35`) |
| `01:00` (ohne `.0`) | reicht automatisch alle Funktionen des Geräts durch (z. B. GPU + Audio-Device) – entspricht der "All Functions"-Checkbox im Web UI |

## Schritt 6: NVIDIA Code 43 vermeiden (Windows-Gast)

NVIDIA-Treiber verweigern in manchen Windows-VMs absichtlich den Dienst, wenn sie eine virtualisierte Umgebung erkennen:

```
# In der VM-Konfiguration ergänzen:
cpu: host,hidden=1,flags=+pcid
args: -cpu 'host,+kvm_pv_unhalt,+kvm_pv_eoi,hv_vendor_id=NV43FIX,kvm=off'
```

---

## SR-IOV als Alternative

Für Netzwerkkarten und manche GPUs unterstützt die Hardware selbst Virtualisierung (Single-Root I/O Virtualization): ein physisches Gerät stellt mehrere Virtual Functions (VFs) bereit, die einzeln an unterschiedliche VMs durchgereicht werden können – mit besserer Performance/Latenz als reine Software-Virtualisierung und ohne das Gerät exklusiv an eine VM zu binden.

---

## Funktionstest

```bash
# Innerhalb der VM nach dem Boot:
lspci | grep -i nvidia
nvidia-smi   # bei korrekt installiertem Treiber im Gast
```

- **Boot-Test:** VM startet ohne Fehler, kein "Out of Memory"/BAR-Fehler bei großem VRAM
- **Bildausgabe:** bei `x-vga=1` Monitor direkt an die durchgereichte Karte anschließen – die Proxmox-Konsole zeigt diese VM danach nicht mehr an
- **Reboot-Test:** VM innerhalb der VM neu starten, bevor produktiv gegangen wird – manche ältere GPUs (z. B. AMD Polaris/Vega) initialisieren sich nach einem Soft-Reset nicht zuverlässig neu

---

## Best Practices

✅ `q35` + OVMF + PCIe als Standard-Kombination für Passthrough-VMs
✅ IOMMU-Gruppen vor dem Hardware-Kauf prüfen, nicht erst danach
✅ Zweite GPU/iGPU für den Host-Konsolenzugriff einplanen
✅ Backups bei VMs mit Passthrough nur im ausgeschalteten Zustand – Live-Snapshots erfassen den physischen Gerätezustand nicht
✅ VM-RAM ohne Ballooning/KSM bei Passthrough – der gesamte Speicher muss gepinnt sein

⚠️ ACS-Override-Patch nur bewusst und mit Verständnis der Isolations-Einbußen einsetzen
⚠️ Bei mehreren identischen GPUs: Bindung per PCI-Adresse statt Vendor:Device-ID, sonst werden beide gleichzeitig gegriffen
⚠️ Aktuelle Mainboard-BIOS-Version verwenden – IOMMU-Gruppierung wird oft in späteren BIOS-Versionen verbessert

❌ PCI-Passthrough auf VMs mit hohem Verfügbarkeitsanspruch ohne Akzeptanz des Backup-/Live-Migration-Verlusts einsetzen – Passthrough-VMs sind nicht live-migrierbar
❌ Erwarten, dass Live-Snapshots bei aktivem Passthrough konsistent funktionieren

---

## Häufige Fehler & Lösungen

| Problem | Ursache | Lösung |
|---|---|---|
| VM startet nicht, `QEMU exited with code 1` | VM-Speicher nicht vollständig pinnbar (Ballooning/KSM aktiv) | RAM reduzieren oder Ballooning deaktivieren, genug freien Host-RAM sicherstellen |
| "This device cannot find enough free resources (Code 12)" | OVMF-PCI-MMIO-Fenster zu klein für großes VRAM | `cpu: host` setzen oder `args: -fw_cfg name=opt/ovmf/X-PciMmio64Mb,string=65536` |
| Schwarzes Bild trotz erfolgreicher Bindung | Falscher Monitor-Anschluss (Host- statt VM-GPU) oder Anzeigeeinstellung im Gast | Monitor an die durchgereichte Karte anschließen, im Gast Windows-Taste+P prüfen |
| NVIDIA Code 43 im Geräte-Manager | Treiber erkennt virtualisierte Umgebung | CPU-Hiding-Flags aus Schritt 6 setzen |
| `hostpci0`-Zeile wird beim Start lautlos verworfen | Ungültige Syntax im Config-Schema (z. B. veraltete `iommufd`-Optionen) | Konfiguration auf aktuelle PVE-9-Syntax prüfen, `qm config <vmid>` kontrollieren |

---

## Weiterführende Links

| Ressource | URL |
|---|---|
| PCI(e) Passthrough (offiziell) | https://pve.proxmox.com/wiki/PCI(e)_Passthrough |
| PCI Passthrough – Troubleshooting | https://pve.proxmox.com/wiki/PCI_Passthrough |
