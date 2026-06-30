<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: Proxmox - Installationen -->
<!-- Title: Proxmox - PVE Erstinstallation (Stand PVE 9.1) -->
<!-- Label: pve-installation, checkliste -->

# PVE Erstinstallation (Stand PVE 9.1)

> **Version:** Proxmox VE 9.x (Debian 13 Trixie) · Hinweise für PVE 8.x (Debian 12 Bookworm) jeweils markiert
> Quelle: adaptiert aus Kapitel 01, Abschnitt 1–3. **Vor Sync gegen bestehenden Confluence-Stand prüfen.**
> Verwandte Seiten: [Post installation check](post-installation-check.md) · [Single Node - Netzwerk](../netzwerk-firewall-konfigurationen/single-node-netzwerk.md)

---

## Wann anwenden?

- Neuer Server wird mit Proxmox VE frisch installiert
- Neuer Cluster-Node wird vorbereitet, **bevor** er einem Cluster beitritt

> Nicht abgedeckt: Post-Install-Schritte → [Post installation check](post-installation-check.md) · Cluster-Setup → Netzwerk-Seiten unter "Netzwerk- & Firewall-Konfigurationen"

---

## Überblick

Proxmox VE wird direkt auf Bare-Metal-Hardware installiert (Typ-1-Hypervisor). Die Installation läuft in zwei Phasen: **Installer** (Disk, Standort, Passwort, Netzwerk) und **Web-UI Setup** (Netzwerk anpassen, Post-Install, Repositories).

> **Hyper-V-Vergleich:** PVE entspricht dem Hyper-V Server (Core) – nur die Hypervisor-Rolle, Web-UI statt RSAT. Die Linux Bridge `vmbr0` ist der virtuelle Switch.

---

## Voraussetzungen

- Statische IP, Subnetz, Gateway, DNS bereit
- Hostname im FQDN-Format definiert (z. B. `pve01.kunde.local`)
- Passwort für Bitwarden vorbereitet
- iDRAC/iLO vorab konfiguriert und erreichbar

---

## 1. Hardware-Vorprüfung

### BIOS/UEFI-Einstellungen

| Einstellung | Sollwert | Warum? |
|---|---|---|
| Intel VT-x / AMD-V | Aktiviert | Pflicht für KVM/QEMU |
| IOMMU (VT-d / AMD-Vi) | Aktiviert | PCI-Passthrough |
| Secure Boot | Deaktiviert | Verhindert Proxmox-Kernelstart |
| Hyperthreading | Aktiviert | CPU-Durchsatz für VMs |

> ⚠️ Dell iDRAC / HP iLO / Supermicro IPMI vor der PVE-Installation konfigurieren – danach ist physischer Zugang meist nur noch darüber sinnvoll möglich.

### RAID-Controller → HBA-IT-Mode (bei ZFS zwingend)

> ❌ Hardware-RAID-Controller sind mit ZFS inkompatibel – ZFS braucht direkten Disk-Zugriff für Checksummen, S.M.A.R.T. und Self-Healing.

```bash
lspci -nn | grep -iE "raid|sas|storage"
```

### NIC-Kompatibilität

| NIC | Treiber | Empfehlung |
|---|---|---|
| Intel i210, i350 (1G) | `igb` | ✅ Erste Wahl |
| Intel X540, X550 (10G) | `ixgbe` | ✅ Empfohlen |
| Intel X710 (10/25G) | `i40e` | ✅ SDN/VLAN |
| Broadcom BCM5719+ | `bnx2x` | ⚠️ Vorab validieren |
| Realtek RTL8111/8125 | `r8169` | ⚠️ Nur Management |

---

## 2. Installation

```bash
# ISO-Download: https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso
# Linux USB erstellen:
sudo dd if=proxmox-ve_9.2-1.iso of=/dev/sdX bs=4M status=progress conv=fsync
# Windows: Rufus im DD-Modus (nicht ISO-Modus!) oder Ventoy
```

**Installer-Typ:** Graphical (Standard) oder Terminal UI (Fallback bei Darstellungsproblemen).

**Dateisystem-Entscheidung** (Details → [LVM & ZFS](../lvm-zfs.md)):

| Szenario | Dateisystem |
|---|---|
| Einzelne OS-Disk | ext4 |
| Zwei gleichartige Disks | ZFS Mirror |
| Drei oder mehr Disks | ZFS RAIDZ-1/2 |

**Standort, Zeitzone, Tastatur:**
```
Country: Germany | Timezone: Europe/Berlin | Keyboard: German
```

**Root-Passwort & E-Mail:** sicheres Passwort vergeben, sofort in Bitwarden hinterlegen; Alert-Adresse eintragen (VIREQ-Empfehlung: zentrale `proxmox-alerts@vireq.com` statt individueller Techniker-Adressen).

**Management-Netzwerk:** eine NIC für initialen Web-UI-Zugriff wählen, Hostname im FQDN-Format, **IPv4 oder IPv6 – nicht beide gleichzeitig**.

**Abschluss:** "Automatically reboot after successful installation" ankreuzen → Install → USB-Stick nach Neustart entfernen.

---

## 3. Web-UI Setup

```
https://<IP-Adresse>:8006
Benutzer: root | Realm: Linux PAM standard authentication
```

> Zertifikatwarnung beim ersten Aufruf ist normal (selbstsigniertes Zertifikat).

**Netzwerk nachkonfigurieren (bei zwei NICs, Redundanz via Bond):**

```
PVE Web-UI: Node → Network → Create → Linux Bond
Name:        bond0
Bond Mode:   active-backup   ← Standard, kein Switch-Config nötig
Bond slaves: eno1 eno2

Anschließend vmbr0 anpassen: Bridge ports → bond0
```

> ⚠️ Netzwerkänderungen erst mit **Apply Configuration** aktivieren. Bei Remote-Zugriff iDRAC/iLO bereithalten.

---

## Best Practices

✅ NVMe SSD als OS-Disk – ext4 bei Einzeldisk, ZFS Mirror bei zwei Disks
✅ Passwort sofort in Bitwarden hinterlegen
✅ Netzwerk: zunächst eine NIC, Bond im Web-UI nachkonfigurieren
✅ iDRAC/iLO vor PVE-Installation konfigurieren

⚠️ IPv4 **oder** IPv6 im Installer angeben – nicht beide
⚠️ Bond active-backup ist der sichere Standard, 802.3ad/LACP nur mit gepflegter Switch-Konfiguration

❌ Hardware-RAID mit ZFS · Realtek für VM-Traffic

---

## Häufige Fehler & Lösungen

**Web-UI nicht erreichbar nach Netzwerk-Änderung**
```
Ursache: Bond-Konfiguration inkorrekt angewendet.
Lösung:  Über iDRAC/iLO einloggen, /etc/network/interfaces prüfen.
```

---

## Weiterführende Links

| Ressource | URL |
|---|---|
| PVE ISO Download | https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso |
| PVE Admin Guide | https://pve.proxmox.com/pve-docs/pve-admin-guide.html |
| HCL (NIC/HBA) | https://pve.proxmox.com/wiki/Hardware_Compatibility_List |
