# Kapitel 1: PVE Grundkonfiguration

> **Version:** Proxmox VE 9.x (Debian 13 Trixie) · Hinweise für PVE 8.x (Debian 12 Bookworm) jeweils markiert  
> **Letzte Aktualisierung:** 2026-06  
> **Verwandte Kapitel:** [Kapitel 2 – Netzwerk & Clustering](02-netzwerk-clustering.md) · [Kapitel 3 – Backup-Strategien](03-backup-strategien.md) · [Kapitel 5 – Sicherheit & Hardening](05-sicherheit-hardening.md)

---

## Wann anwenden?

> - Neuer Server wird mit Proxmox VE frisch installiert
> - Bestehender Node nach Neuinstallation grundkonfiguriert
> - Neuer Cluster-Node wird vorbereitet, **bevor** er dem Cluster beitritt (→ Kapitel 2)
>
> Nicht abgedeckt: Cluster-Setup → Kap. 2 · Backup/PBS → Kap. 3 · SSH/Firewall/2FA → Kap. 5

---

## Überblick

Proxmox VE wird direkt auf bare-metal Hardware installiert (Typ-1-Hypervisor). Die Installation läuft in zwei Phasen:

1. **Installer** – Disk, Standort, Passwort, Netzwerk
2. **Web-UI Setup** – Netzwerk anpassen, Post-Install, Repositories

> **Hyper-V-Vergleich:** PVE entspricht dem Hyper-V Server (Core) – nur die Hypervisor-Rolle, Web-UI statt RSAT. Die Linux Bridge `vmbr0` ist der virtuelle Switch.

---

## Voraussetzungen

- Statische IP, Subnetz, Gateway, DNS bereit
- Hostname im FQDN-Format definiert (z. B. `pve01.kunde.local`)
- Passwort für Bitwarden vorbereitet
- iDRAC / iLO vorab konfiguriert und erreichbar

---

## 1. Hardware-Vorprüfung

### 1.1 BIOS/UEFI-Einstellungen

| Einstellung | Sollwert | Warum? |
|-------------|----------|--------|
| Intel VT-x / AMD-V | **Aktiviert** | Pflicht für KVM/QEMU |
| IOMMU (VT-d / AMD-Vi) | **Aktiviert** | PCI-Passthrough |
| Secure Boot | **Deaktiviert** | Verhindert Proxmox-Kernelstart |
| Hyperthreading | **Aktiviert** | CPU-Durchsatz für VMs |
| C-States | Optional deaktivieren | Latenz-sensitive Workloads |

> ⚠️ **Dell iDRAC / HP iLO / Supermicro IPMI:** Vor der PVE-Installation konfigurieren. Nach der Installation ist der physische Zugang meistens nur noch darüber sinnvoll möglich.

### 1.2 RAID-Controller → HBA IT-Mode (bei ZFS zwingend)

> ❌ Hardware-RAID-Controller sind mit ZFS inkompatibel — ZFS braucht direkten Disk-Zugriff für Checksummen, S.M.A.R.T. und Self-Healing. Nur relevant wenn ZFS verwendet wird (→ Abschnitt 2.3).

| Controller | Chip | Konvertierungspfad |
|-----------|------|-------------------|
| Dell PERC H200 | LSI SAS2008 | Crossflash auf LSI 9211-8i IT-Mode |
| Dell PERC H310 | LSI SAS2008 | Crossflash auf LSI 9211-8i IT-Mode |
| Dell PERC H330 | LSI SAS3008 | Spezielle Brücke erforderlich |
| Dell PERC H730/H740 | Broadcom SAS3 | Kein IT-Mode möglich – Austausch |
| LSI SAS2008 / SAS2308 | – | Direkt IT-Mode flashbar |
| Broadcom / Avago HBA | – | Meist bereits IT-Mode |

```bash
lspci -nn | grep -iE "raid|sas|storage"
# [1000:xxxx] = LSI/Broadcom · [1028:xxxx] = Dell
```

### 1.3 NIC-Kompatibilität

| NIC | Treiber | Empfehlung |
|-----|---------|----------|
| Intel i210, i350 (1G) | `igb` | ✅ Erste Wahl |
| Intel X540, X550 (10G) | `ixgbe` | ✅ Empfohlen |
| Intel X710 (10/25G) | `i40e` | ✅ SDN/VLAN |
| Broadcom BCM5719+ | `bnx2x` | ⚠️ Vorab validieren |
| Mellanox ConnectX-4/5 | `mlx5_en` | ✅ 25G/100G |
| Realtek RTL8111/8125 | `r8169` | ⚠️ Nur Management |

---

## 2. Installation

### 2.1 ISO herunterladen und USB erstellen

```
ISO-Download: https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso
```

```bash
# Linux:
sudo dd if=proxmox-ve_9.2-1.iso of=/dev/sdX bs=4M status=progress conv=fsync
```

```
# Windows: Rufus → DD-Modus (NICHT ISO-Modus!) · Alternativ: Ventoy
```

### 2.2 Installer-Typ wählen

Nach dem Boot vom USB erscheinen zwei Optionen:

| Option | Wann verwenden? |
|--------|----------------|
| **Install Proxmox VE (Graphical)** | Standard — bei normaler Hardware |
| **Install Proxmox VE (Terminal UI)** | Fallback — bei sehr alter oder sehr neuer Hardware, wenn der Graphical-Installer nicht korrekt dargestellt wird |

### 2.3 EULA akzeptieren

### 2.4 Zieldisk auswählen

**Empfehlung:** NVMe SSD als OS-Disk bevorzugen.

**Dateisystem-Entscheidung:**

| Szenario | Dateisystem | Begründung |
|----------|-------------|-----------|
| **Einzelne OS-Disk** | **ext4** | ZFS auf einer Disk bietet keine Redundanz, belegt aber RAM für ARC — ext4 ist effizienter |
| **Zwei gleichartige Disks** | **ZFS Mirror** | Redundanz für OS und VM-Storage in einem, Self-Healing, schnelle Snapshots |
| **Drei oder mehr Disks** | **ZFS RAIDZ-1/2** | RAIDZ-1 ab 3 Disks, RAIDZ-2 ab 4 Disks |

> 💡 **VM-Storage separat:** Auch wenn die OS-Disk ext4 läuft, kann ein dedizierter ZFS-Pool für VM-Disks nachträglich eingerichtet werden (→ Abschnitt 5.2). Das ist die empfohlene Kombination bei nur einer NVMe OS-Disk und separaten Storage-Disks.

> 💡 **ashift bei ZFS:** Moderne SSDs/NVMe → `ashift=12`. Dieser Wert lässt sich nachträglich **nicht** ändern.

### 2.5 Standort, Zeitzone & Tastatur

```
Country:   Germany
Timezone:  Europe/Berlin
Keyboard:  German
```

### 2.6 Root-Passwort & E-Mail

- Sicheres Passwort vergeben und **sofort in Bitwarden hinterlegen**
- E-Mail-Adresse für System-Alerts eintragen

> 💡 **Zentrale Alert-Adresse:** Für Kundenprojekte empfiehlt sich eine dedizierte Postfach-Adresse (z. B. `proxmox-alerts@vireq.com`) statt individueller Techniker-Adressen — so gehen keine Benachrichtigungen verloren wenn jemand im Urlaub ist.

### 2.7 Management-Netzwerk konfigurieren

- **Interface:** Zunächst eine Netzwerkkarte wählen, um nach der Installation Zugriff auf die Web-UI zu haben. Weitere NICs werden im Web-UI nachkonfiguriert (→ Abschnitt 3.2)
- **Hostname:** Im FQDN-Format angeben: `pve01.kunde.local`
- **IP-Adresse:** Entweder IPv4 **oder** IPv6 angeben — **nicht beide gleichzeitig**
- Alle Netzwerkeinstellungen können nach der Installation im Web-UI angepasst werden

### 2.8 Zusammenfassung & Installation

- Alle Einstellungen prüfen
- ✅ **"Automatically reboot after successful installation"** ankreuzen
- → **Install** klicken, USB-Stick nach dem Neustart entfernen

---

## 3. Web-UI Setup

### 3.1 Anmelden

```
https://<IP-Adresse>:8006
Benutzer: root
Realm:    Linux PAM standard authentication
```

> Der Browser zeigt eine Zertifikatwarnung (selbstsigniertes Zertifikat) — das ist bei der Erstinstallation normal. Eigenes Zertifikat → Kapitel 5.

### 3.2 Netzwerk konfigurieren

**Einzelne Netzwerkkarte:** Keine Änderungen notwendig — `vmbr0` läuft bereits auf der gewählten NIC.

**Zwei Netzwerkkarten (Redundanz via Bond):**

```
PVE Web-UI: Node → Network → Create → Linux Bond

Name:        bond0
Bond Mode:   active-backup   ← Standard, funktioniert ohne Switch-Konfiguration
Bond slaves: eno1 eno2        ← beide physischen NICs als Slaves
```

```
Anschließend vmbr0 anpassen:
Bridge ports: bond0           ← statt der einzelnen NIC
```

> ⚠️ Netzwerkänderungen erst mit **Apply Configuration** aktivieren. Bei Remote-Zugriff: iDRAC/iLO bereithalten.

> 💡 **Weitere Bond-Modi** (z. B. 802.3ad/LACP für höhere Bandbreite) erfordern Switch-seitige Konfiguration. Übersicht: [PVE Network Configuration – Linux Bond](https://pve.proxmox.com/wiki/Network_Configuration#sysadmin_network_bond)

---

## 4. Post-Install-Konfiguration

```bash
ssh root@192.168.1.10
```

### 4.1 Systemzustand prüfen

```bash
pveversion -v
ip addr show
ping -c 3 8.8.8.8
```

### 4.2 Repository umstellen (No-Subscription)

> Ohne Umstellung schlägt `apt update` mit `401 Unauthorized` fehl.

```bash
# Enterprise-Repo deaktivieren (PVE 9.x / Trixie):
cat > /etc/apt/sources.list.d/pve-enterprise.list << 'EOF'
# deb https://enterprise.proxmox.com/debian/pve trixie pve-enterprise
EOF

cat > /etc/apt/sources.list.d/ceph.list << 'EOF'
# deb https://enterprise.proxmox.com/debian/ceph-squid trixie enterprise
EOF

# No-Subscription-Repo aktivieren:
cat >> /etc/apt/sources.list << 'EOF'

deb http://download.proxmox.com/debian/pve trixie pve-no-subscription
EOF

apt update
```

> **PVE 8.x (Bookworm):** `trixie` → `bookworm`, `ceph-squid` → `ceph-quincy`

### 4.3 System aktualisieren

```bash
apt full-upgrade -y
reboot
pveversion -v
```

### 4.4 Hosts-Datei konfigurieren ⚠️ (Cluster-kritisch)

> Hostname **muss** auf Management-IP zeigen — nie auf `127.0.x.x`. Falscher Eintrag = Cluster-Join schlägt fehl.

```bash
cat > /etc/hosts << 'EOF'
127.0.0.1       localhost
192.168.1.10    pve01.kunde.local pve01

::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
EOF

hostname --fqdn
# Muss lauten: pve01.kunde.local
```

### 4.5 Zeitsynchronisation

```bash
chronyc tracking
timedatectl set-timezone Europe/Berlin

# Eigener NTP-Server (optional, in /etc/chrony/chrony.conf):
server ntp.vireq.local iburst prefer
systemctl restart chrony
```

### 4.6 E-Mail-Benachrichtigungen (SMTP)

**Via Web-UI (PVE 9.x – empfohlen):**
```
Datacenter → Notifications → Add → SMTP
Host: smtp.office365.com · Port: 587 · Mode: STARTTLS
Username/Passwort · From/To eintragen
```

**Via CLI (Postfix + Office 365):**
```bash
apt install -y libsasl2-modules

cat > /etc/postfix/sasl_passwd << 'EOF'
[smtp.office365.com]:587 proxmox-alerts@vireq.com:PASSWORT
EOF
postmap /etc/postfix/sasl_passwd
chmod 600 /etc/postfix/sasl_passwd*

cat >> /etc/postfix/main.cf << 'EOF'
relayhost = [smtp.office365.com]:587
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_tls_security_level = encrypt
EOF

systemctl restart postfix
echo "Test" | mail -s "[PVE] Test $(hostname)" proxmox-alerts@vireq.com
```

### 4.7 ZFS ARC begrenzen (nur bei ZFS-Installation)

> Nicht nötig bei ext4-Installation.

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

### 4.8 Subscription-Warnung entfernen (optional)

```bash
sed -i.bak "s/if (data.status !== 'Active')/if (false)/g" \
  /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
systemctl restart pveproxy

# APT-Hook – automatisch nach Updates:
cat > /etc/apt/apt.conf.d/99-proxmox-nosubwarn << 'EOF'
DPkg::Post-Invoke {
  "sed -i.bak 's/if (data.status !== .Active.)/if (false)/g' \
    /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js \
    && systemctl restart pveproxy || true;";
};
EOF
```

---

## 5. Storage-Konfiguration

### 5.1 Standard-Storage nach Installation

| Storage | Typ | Inhalte | Bei ext4-OS |
|---------|-----|---------|------------|
| `local` | Directory `/var/lib/vz` | ISOs, Templates, Backups | ✅ vorhanden |
| `local-lvm` | LVM-Thin | VM-Disks, Container | ✅ vorhanden |

> Bei **ZFS-OS-Installation** gibt es kein `local-lvm` — VM-Storage läuft direkt auf ZFS-Datasets.

### 5.2 Separaten ZFS-Pool für VMs anlegen (empfohlen bei ext4-OS)

> Immer `/dev/disk/by-id/` verwenden — nie `/dev/sdX`!

```bash
# Verfügbare Disks:
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
# Web-UI: Datacenter → Storage → Add → ZFS
```

### 5.3 NFS-Share einbinden

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

✅ NVMe SSD als OS-Disk — ext4 bei einzelner Disk, ZFS Mirror bei zwei Disks  
✅ VM-Storage immer auf separatem ZFS-Pool, auch wenn OS ext4 nutzt  
✅ Passwort sofort in Bitwarden hinterlegen  
✅ Netzwerk: zunächst eine NIC, Bond im Web-UI nachkonfigurieren  
✅ `apt full-upgrade` (nicht `apt upgrade`) — Hosts-Datei vor Cluster-Join prüfen  
✅ iDRAC/iLO vor PVE-Installation konfigurieren

⚠️ Bond active-backup ist der sichere Standard (kein Switch-Config nötig) — 802.3ad/LACP nur mit gepflegter Switch-Konfiguration  
⚠️ IPv4 **oder** IPv6 im Installer angeben — nicht beide  
⚠️ Subscription-Warnung-Patch wird nach Updates zurückgesetzt → APT-Hook nutzen

❌ Hardware-RAID mit ZFS — Realtek für VM-Traffic — `127.0.1.1` in `/etc/hosts`

---

## Häufige Fehler & Lösungen

**`401 Unauthorized` bei apt update**
```
Ursache: Enterprise-Repo aktiv, keine Subscription.
Lösung:  Abschnitt 4.2 – No-Subscription-Repo aktivieren.
```

**`hostname --fqdn` gibt `localdomain` zurück**
```
Ursache: /etc/hosts falsch konfiguriert.
Lösung:  Abschnitt 4.4 – Management-IP eintragen.
```

**Web-UI nicht erreichbar nach Netzwerk-Änderung**
```
Ursache: Bond-Konfiguration inkorrekt angewendet.
Lösung:  Über iDRAC/iLO einloggen, /etc/network/interfaces prüfen.
```

**ZFS ARC verdrängt VM-RAM**
```
Ursache: Kein ARC-Limit gesetzt (nur bei ZFS-Installation relevant).
Lösung:  Abschnitt 4.7 – zfs_arc_max setzen.
```

---

## Weiterführende Links

| Ressource | URL |
|-----------|-----|
| PVE ISO Download | https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso |
| PVE Admin Guide | https://pve.proxmox.com/pve-docs/pve-admin-guide.html |
| PVE Network Config (Bond-Modi) | https://pve.proxmox.com/wiki/Network_Configuration#sysadmin_network_bond |
| OpenZFS Docs | https://openzfs.github.io/openzfs-docs/ |
| HCL (NIC/HBA) | https://pve.proxmox.com/wiki/Hardware_Compatibility_List |
| PERC H310 IT-Mode | https://fohdeesha.com/docs/perc.html |

---

**→ Nächstes Kapitel:** [Kapitel 2 – Netzwerk & Clustering](02-netzwerk-clustering.md)  
**← Übersicht:** [README.md](../README.md)
