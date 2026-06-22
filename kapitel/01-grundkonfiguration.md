# Kapitel 1: PVE Grundkonfiguration

> **Version:** Proxmox VE 9.x (Debian 13 Trixie) · Hinweise für PVE 8.x (Debian 12 Bookworm) jeweils markiert  
> **Letzte Aktualisierung:** 2026-06  
> **Verwandte Kapitel:** [Kapitel 2 – Netzwerk & Clustering](02-netzwerk-clustering.md) · [Kapitel 3 – Backup-Strategien](03-backup-strategien.md) · [Kapitel 5 – Sicherheit & Hardening](05-sicherheit-hardening.md)

---

## Wann anwenden?

> **Dieses Kapitel ist relevant, wenn:**
> - ein neuer Server mit Proxmox VE frisch installiert werden soll,
> - ein bestehender Node nach einer Neuinstallation grundkonfiguriert werden muss,
> - ein neuer Cluster-Node vorbereitet wird, **bevor** er dem Cluster beitritt (→ Kapitel 2).
>
> **Dieses Kapitel deckt nicht ab:**
> - Cluster-Konfiguration und Netzwerkdesign → Kapitel 2
> - Backup-Einrichtung und PBS-Integration → Kapitel 3
> - Firewall, SSH-Hardening, 2FA → Kapitel 5

---

## Überblick

Proxmox VE wird direkt auf bare-metal Hardware installiert (Typ-1-Hypervisor). Die Grundkonfiguration umfasst vier Phasen:

1. **Hardware-Vorprüfung** – BIOS, RAID-Controller und NIC-Kompatibilität sicherstellen
2. **Installation** – Dateisystem-Entscheidung, ISO-Boot, Ersteinrichtung
3. **Post-Install** – Repositories, Updates, Hosts-Datei, Zeitsync, E-Mail, ZFS-ARC
4. **Storage & Netzwerk** – Speicher und Netzwerkbrücken für VMs vorbereiten

> **Hyper-V-Vergleich:** PVE entspricht dem Hyper-V Server (Core) – kein vollständiges Windows, nur die Hypervisor-Rolle mit Web-UI statt RSAT. Die Linux Bridge `vmbr0` entspricht dem virtuellen Switch in Hyper-V.

---

## Voraussetzungen

**Hardware:**
- 64-Bit-CPU mit Intel VT-x oder AMD-V
- Minimum 8 GB RAM (2 GB OS + 1 GB/TB ZFS ARC + VM-RAM)
- Minimum 2 × SSD/HDD für ZFS Mirror (Produktion)
- Mindestens 1 × Intel-NIC

**Informationen bereithalten:**
- Statische IP, Subnetzmaske, Gateway, DNS
- Hostname und FQDN (z. B. `pve01.firma.local`)
- Root-Passwort, E-Mail für Alerts

**Entscheidungen vorab:**
- [ ] ZFS (Produktion) oder ext4/LVM-Thin (Testlab)?
- [ ] Hardware-RAID-Controller verbaut? → HBA IT-Mode erforderlich (Abschnitt 1.2)
- [ ] Broadcom-NIC? → vorab validieren (Abschnitt 1.3)

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

> ⚠️ **Dell iDRAC / HP iLO:** Remote-Management vor PVE-Installation konfigurieren.

### 1.2 RAID-Controller → HBA IT-Mode (kritisch für ZFS)

> ❌ **Hardware-RAID-Controller sind mit ZFS inkompatibel.** ZFS benötigt direkten Zugriff auf jede physische Disk für Checksummen, S.M.A.R.T. und Self-Healing. Hardware-RAID versteckt die Disks hinter einer logischen Einheit – das führt bei Fehlerfällen zu Datenverlust.

**Lösung: Controller auf HBA IT-Mode umflashen.**

| Controller | Chip | Konvertierungspfad |
|-----------|------|-------------------|
| Dell PERC H200 | LSI SAS2008 | Crossflash auf LSI 9211-8i IT-Mode |
| Dell PERC H310 | LSI SAS2008 | Crossflash auf LSI 9211-8i IT-Mode |
| Dell PERC H330 | LSI SAS3008 | Spezielle Brücke erforderlich |
| Dell PERC H730/H740 | Broadcom SAS3 | Kein IT-Mode möglich – Austausch |
| LSI SAS2008 / SAS2308 | – | Direkt IT-Mode flashbar |
| Broadcom / Avago HBA | – | Meist bereits IT-Mode |

```bash
# Controller identifizieren:
lspci -nn | grep -iE "raid|sas|storage"
# [1000:xxxx] = LSI/Broadcom, [1028:xxxx] = Dell
```

### 1.3 NIC-Kompatibilität

| NIC-Familie | Treiber | Empfehlung |
|------------|---------|----------|
| Intel i210, i350 (1G) | `igb` | ✅ Erste Wahl |
| Intel X540, X550 (10G) | `ixgbe` | ✅ Empfohlen |
| Intel X710 (10/25G) | `i40e` | ✅ Gut für SDN/VLAN |
| Broadcom BCM5719+ | `bnx2x` | ⚠️ Vorab validieren |
| Mellanox ConnectX-4/5 | `mlx5_en` | ✅ 25G/100G |
| Realtek RTL8111/8125 | `r8169` | ⚠️ Nur Management |

---

## 2. Proxmox VE Installation

### 2.1 ISO und USB

```
https://www.proxmox.com/de/downloads/proxmox-virtual-environment/iso
```

```bash
# Linux (X = USB-Gerät, kein Suffix):
sudo dd if=proxmox-ve_9.2-1.iso of=/dev/sdX bs=4M status=progress conv=fsync
```

```
# Windows: Rufus → DD-Modus (NICHT ISO-Modus!)
# Alternativ: Ventoy
```

### 2.2 Dateisystem-Entscheidung

| Kriterium | ZFS (Mirror/RAIDZ) | ext4/LVM-Thin |
|-----------|-------------------|---------------|
| Datensicherheit | ✅ Checksummen, Self-Healing | Standard |
| RAM-Bedarf | +1 GB/TB (ARC) | Minimal |
| Snapshots | ✅ Nativ, schnell | Via LVM |
| Kompression | ✅ lz4, transparent | Nein |
| **Empfehlung** | **Produktion** | **Testlab / <16 GB RAM** |

| ZFS-Konfig | Disks | Ausfall toleriert | Empfehlung |
|-----------|-------|-----------------|----------|
| Mirror | 2 | 1 Disk | ✅ Standard |
| RAIDZ-1 | ≥3 | 1 Disk | 3–5 Disks |
| RAIDZ-2 | ≥4 | 2 Disks | ✅ Ab 4 Disks |

> 💡 **ashift:** Moderne SSDs/HDDs → `ashift=12`. Nicht nachträglich änderbar!

### 2.3 Installationsschritte

1. Von USB booten → **"Install Proxmox VE (Graphical)"**
2. EULA akzeptieren
3. Target Disk(s): ZFS Mirror, `ashift=12`
4. Location: Germany · Europe/Berlin · Keyboard: de
5. Passwort + E-Mail setzen
6. Netzwerk: Management-NIC, Hostname `pve01.firma.local`, IP/Gateway/DNS
7. → **Install**, USB entfernen, Neustart

```
https://192.168.1.10:8006  →  root / Linux PAM
```

---

## 3. Post-Install-Konfiguration

```bash
ssh root@192.168.1.10
```

### 3.1 Systemzustand prüfen

```bash
pveversion -v
zpool status && zpool list
ip addr show
ping -c 3 8.8.8.8
```

### 3.2 Repository umstellen (No-Subscription)

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

> **PVE 8.x:** `trixie` → `bookworm`, `ceph-squid` → `ceph-quincy`

### 3.3 System aktualisieren

```bash
apt full-upgrade -y
reboot
pveversion -v
```

### 3.4 Hosts-Datei konfigurieren ⚠️ (Cluster-kritisch)

> Hostname **muss** auf Management-IP zeigen, nie auf `127.0.x.x`. Falscher Eintrag = Cluster-Join schlägt fehl.

```bash
cat > /etc/hosts << 'EOF'
127.0.0.1       localhost
192.168.1.10    pve01.firma.local pve01

::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
EOF

hostname --fqdn
# Muss lauten: pve01.firma.local
```

### 3.5 Zeitsynchronisation

```bash
chronyc tracking
timedatectl set-timezone Europe/Berlin

# Eigenen NTP-Server (in /etc/chrony/chrony.conf):
server ntp.firma.local iburst prefer
systemctl restart chrony
```

### 3.6 E-Mail-Benachrichtigungen

**Via Web-UI (PVE 9.x):** `Datacenter → Notifications → Add → SMTP`

**Via CLI (Office 365):**
```bash
apt install -y libsasl2-modules

cat > /etc/postfix/sasl_passwd << 'EOF'
[smtp.office365.com]:587 alert@firma.de:PASSWORT
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
echo "Test" | mail -s "[PVE] Test" admin@firma.de
```

### 3.7 ZFS ARC begrenzen

**Formel:** `RAM = 2 GB OS + 1 GB/TB ARC + VM-RAM`

```bash
# Aktuellen Verbrauch prüfen:
awk '/^size/{printf "ARC: %.1f GB\n", $3/1073741824}' /proc/spl/kstat/zfs/arcstats

# ARC auf 4 GB begrenzen (dauerhaft):
cat > /etc/modprobe.d/zfs.conf << 'EOF'
options zfs zfs_arc_max=4294967296
options zfs zfs_arc_min=1073741824
EOF
update-initramfs -u

# Sofort (temporär):
echo 4294967296 > /sys/module/zfs/parameters/zfs_arc_max
```

### 3.8 Subscription-Warnung entfernen (optional)

```bash
sed -i.bak "s/if (data.status !== 'Active')/if (false)/g" \
  /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
systemctl restart pveproxy

# APT-Hook (automatisch nach Updates):
cat > /etc/apt/apt.conf.d/99-proxmox-nosubwarn << 'EOF'
DPkg::Post-Invoke {
  "sed -i.bak 's/if (data.status !== .Active.)/if (false)/g' \
    /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js \
    && systemctl restart pveproxy || true;";
};
EOF
```

---

## 4. Storage-Konfiguration

### 4.1 Standard-Storage

| Name | Typ | Inhalte | Einsatz |
|------|-----|---------|---------|
| `local` | Directory `/var/lib/vz` | ISOs, Templates, Backups | Verwaltung |
| `local-lvm` | LVM-Thin | VM-Disks, Container | VM-Storage |

> ⚠️ Bei ZFS-Root-Installation: kein `local-lvm` – separaten ZFS-Pool anlegen (Abschnitt 4.2).

### 4.2 ZFS-Pool für VMs erstellen

> Immer `/dev/disk/by-id/` verwenden – nie `/dev/sdX`!

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,TYPE
ls -l /dev/disk/by-id/ | grep -v part

zpool create -o ashift=12 vmpool mirror \
  /dev/disk/by-id/DISK1-SERIAL \
  /dev/disk/by-id/DISK2-SERIAL

zfs set compression=lz4 vmpool
zpool status vmpool

pvesm add zfspool vmpool --pool vmpool --content images,rootdir
# Web-UI: Datacenter → Storage → Add → ZFS
```

### 4.3 NFS-Share einbinden

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

## 5. Grundlegende Netzwerkkonfiguration

### 5.1 Linux Bridge verstehen

```
LAN
 │
[eno1]      ← Physische NIC (kein IP)
 │
[vmbr0]     ← Bridge (trägt Management-IP)
 ├ VM 100
 ├ VM 101
 └ CT 200
```

> **Hyper-V:** `vmbr0` = External Virtual Switch. NIC ist Uplink ohne eigene IP.

### 5.2 VLAN-fähige Bridge (`/etc/network/interfaces`)

```
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.1.10/24
    gateway 192.168.1.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094

dns-nameservers 192.168.1.1
```

```bash
ifreload -a
ip addr show vmbr0
```

---

## Best Practices

✅ ZFS Mirror/RAIDZ für Produktionssysteme — immer `/dev/disk/by-id/`  
✅ ARC begrenzen bevor VMs starten — `apt full-upgrade` (nicht `upgrade`)  
✅ Hosts-Datei sofort prüfen — E-Mail vor der ersten VM einrichten  
✅ iDRAC/iLO vor PVE-Installation konfigurieren

⚠️ Subscription-Warnung-Patch wird nach Updates zurückgesetzt → APT-Hook nutzen  
⚠️ SSH-Root-Login nach Grundkonfiguration absichern (→ Kapitel 5)

❌ Hardware-RAID mit ZFS — Realtek für VM-Traffic — `127.0.1.1` in `/etc/hosts`

---

## Häufige Fehler & Lösungen

**`401 Unauthorized` bei apt update**
```
Ursache: Enterprise-Repo aktiv, keine Subscription.
Lösung:  Abschnitt 3.2 – No-Subscription-Repo aktivieren.
```

**`hostname --fqdn` gibt `localdomain` zurück**
```
Ursache: /etc/hosts falsch konfiguriert (127.0.1.1).
Lösung:  Abschnitt 3.4 – Management-IP eintragen.
```

**ZFS ARC verdrängt VM-RAM, VMs swappen**
```
Ursache: Kein ARC-Limit gesetzt.
Lösung:  Abschnitt 3.7 – zfs_arc_max setzen.
```

**`zpool status` zeigt DEGRADED nach Neustart**
```
Ursache: Pool mit /dev/sdX erstellt, Buchstabe geändert.
Lösung:  zpool export vmpool && zpool import vmpool
```

---

## Weiterführende Links

| Ressource | URL |
|-----------|-----|
| PVE Admin Guide | https://pve.proxmox.com/pve-docs/pve-admin-guide.html |
| PVE Wiki | https://pve.proxmox.com/wiki/Main_Page |
| OpenZFS Docs | https://openzfs.github.io/openzfs-docs/ |
| HCL (NIC/HBA) | https://pve.proxmox.com/wiki/Hardware_Compatibility_List |
| PERC H310 IT-Mode | https://fohdeesha.com/docs/perc.html |

---

**→ Nächstes Kapitel:** [Kapitel 2 – Netzwerk & Clustering](02-netzwerk-clustering.md)  
**← Übersicht:** [README.md](../README.md)
