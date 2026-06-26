<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: PVE – Best Practices -->
<!-- Title: PVE – Netzwerk: Software Defined Networking (SDN) -->
<!-- Label: pve-netzwerk, best-practice, referenz, pve-cluster -->

# Kapitel 08: Software Defined Networking (SDN)

> **PVE 9.x (Debian 13 Trixie)** · Letzte Aktualisierung: 2026-06  
> Verwandte Kapitel: [02 – Netzwerk & Clustering](02-netzwerk-clustering.md) · [07 – Firewall & IPsets](07-firewall-ipsets.md)

---

## Wann anwenden?

- **Immer beim Cluster-Betrieb** – SDN ist die Best-Practice-Methode für VM-Netzwerke in Proxmox-Clustern
- **Multi-Tenant-Umgebungen** – Trennung von Kunden-/Projektnetzen über VNets statt manueller Bridge-Configs
- **VLAN-Management vereinfachen** – Statt pro Node manuell Bridges konfigurieren: einmal anlegen, automatisch propagiert
- **DHCP/IPAM zentral verwalten** – Subnetze und IP-Vergabe ohne externen DHCP-Server

> **SDN gilt ausschließlich für VM-Traffic** (bond0 / vmbr1). Corosync (ens19), Ceph (bond1) und Management (vmbr0) sind physisch dediziert und werden nicht über SDN verwaltet.

---

## Überblick

Ohne SDN muss jede neue VM-VLAN auf **jedem Cluster-Node einzeln** als Linux-Bridge konfiguriert und `ifreload -a` ausgeführt werden. Proxmox SDN löst dieses Problem: VNets werden einmal im Datacenter angelegt und automatisch auf alle Nodes propagiert.

```
Ohne SDN                                    Mit SDN (VLAN Zone)
──────────────────────────────────          ──────────────────────────────────────────
pve1: vmbr1 + VLAN 100 manuell             Datacenter → SDN → Zone: vlan-zone
pve2: vmbr1 + VLAN 100 manuell               └─ VNet: prod-v100 (VLAN 100)
pve3: vmbr1 + VLAN 100 manuell                    → pve1 ✓  pve2 ✓  pve3 ✓ (auto)

Neue VLAN 200 → 3× Bridge-Konfiguration   Neue VLAN 200 → 1× VNet → Apply → fertig
```

### Hyper-V Vergleich

| Konzept | Hyper-V | Proxmox SDN |
|---------|---------|-------------|
| VM-Netzwerk | Virtuelle Switches (per Host konfiguriert) | VNets (cluster-weit, zentral im Datacenter) |
| VLAN-Zuweisung | Port-VLAN-ID pro VM-NIC | VNet mit VLAN-Tag, VM wählt VNet aus |
| Trunk-Uplink | VM-Switch mit Trunk-Adapter | Zone-Bridge (`vmbr1`) als Trunk |
| DHCP-Server | Separate Windows-DHCP-Rolle | Optional direkt in SDN via dnsmasq |
| Netz-Isolation | Separate vSwitch-Instanzen oder VLANs | Separate VNets pro VLAN / Zone |
| Zentrale Verwaltung | SCVMM / Windows Admin Center | Datacenter → SDN (nativ, kein Add-on) |

---

## SDN-Zonentypen im Vergleich

| Zonentyp | Einsatzgebiet | Komplexität | Switch-Anforderung |
|----------|---------------|-------------|-------------------|
| **VLAN** | Standard-VLAN, Single-Site | ⭐ Niedrig | Trunk-Port mit erlaubten VLANs |
| **QinQ** | VLAN-Stacking, Provider-Netze | ⭐⭐ Mittel | QinQ-fähiger Switch |
| **VXLAN** | L2-Overlay, Multi-Site ohne gemeinsame VLANs | ⭐⭐ Mittel | Nur IP-Routing nötig |
| **EVPN** | Verteiltes L3-Routing mit BGP | ⭐⭐⭐ Hoch | FRR/BGP auf allen Nodes |

> **VIREQ-Standard: VLAN Zone** – Passt zu 98 % der Kunden-Installationen. Gleiche Switches, gleiche VLANs wie bisher – SDN fügt lediglich zentrale Verwaltung hinzu. VXLAN wird relevant bei Multi-Site-Betrieb mit Proxmox Datacenter Manager (Kapitel 04).

---

## Voraussetzungen

- Proxmox VE 9.x Cluster mit aktivem Corosync
- `vmbr1` bereits als VLAN-aware Bridge auf `bond0` konfiguriert (vgl. Kapitel 02)
- Switch-Trunk-Port mit allen benötigten VLANs als erlaubte VLANs konfiguriert
- Package `libpve-network-perl` vorhanden (PVE 9.x Standard)

```bash
# Package-Status prüfen
dpkg -l libpve-network-perl | grep ^ii

# Falls nicht vorhanden
apt update && apt install libpve-network-perl
```

---

## VLAN Zone einrichten

### Schritt 1: Zone erstellen

Die Zone definiert den Typ und die Trunk-Bridge. Eine Zone pro physischem Segment reicht für die meisten Deployments.

**Web UI:** Datacenter → SDN → Zones → Add → VLAN

| Feld | Wert | Erklärung |
|------|------|-----------|
| ID | `vlan-zone` | Interner Bezeichner (keine Sonderzeichen) |
| Bridge | `vmbr1` | Bestehende Trunk-Bridge (bond0-basiert, VLAN-aware) |
| MTU | *(leer)* | Standard 1500 – nur bei VXLAN-Underlay anpassen |

**Per CLI:**
```bash
pvesh create /cluster/sdn/zones \
  --zone vlan-zone \
  --type vlan \
  --bridge vmbr1
```

---

### Schritt 2: VNets erstellen

Jeder VNet entspricht einem VLAN. Naming-Empfehlung: `<funktion>-v<vlan-id>`.

**Web UI:** Datacenter → SDN → VNets → Add

| VNet-Name | VLAN-Tag | Verwendung |
|-----------|----------|------------|
| `prod-v100` | 100 | Produktivnetz / Server-VLAN |
| `mgmt-v200` | 200 | Management-Netz (Switches, iDRAC) |
| `dmz-v300` | 300 | DMZ / extern erreichbare Dienste |
| `backup-v400` | 400 | Backup-Traffic (Hornetsecurity o.ä.) |

**Per CLI:**
```bash
pvesh create /cluster/sdn/vnets --vnet prod-v100   --zone vlan-zone --tag 100
pvesh create /cluster/sdn/vnets --vnet mgmt-v200   --zone vlan-zone --tag 200
pvesh create /cluster/sdn/vnets --vnet dmz-v300    --zone vlan-zone --tag 300
pvesh create /cluster/sdn/vnets --vnet backup-v400 --zone vlan-zone --tag 400
```

---

### Schritt 3: Subnets & DHCP einrichten (optional)

Subnets aktivieren IP-Verwaltung und einen optionalen DHCP-Server direkt in Proxmox. Kein separater DHCP-Server nötig.

```bash
# dnsmasq installieren (einmalig auf allen Nodes)
for node in pve1 pve2 pve3; do
  ssh root@$node "apt install -y dnsmasq"
done

# Subnet für prod-v100 mit DHCP-Range
pvesh create /cluster/sdn/vnets/prod-v100/subnets \
  --subnet 192.168.100.0/24 \
  --type subnet \
  --gateway 192.168.100.1 \
  --dhcp-range start-address=192.168.100.100,end-address=192.168.100.200

# Subnet für mgmt-v200 (ohne DHCP – statische IPs)
pvesh create /cluster/sdn/vnets/mgmt-v200/subnets \
  --subnet 192.168.200.0/24 \
  --type subnet \
  --gateway 192.168.200.1

# Subnet für dmz-v300 mit DHCP
pvesh create /cluster/sdn/vnets/dmz-v300/subnets \
  --subnet 192.168.300.0/24 \
  --type subnet \
  --gateway 192.168.300.1 \
  --dhcp-range start-address=192.168.300.10,end-address=192.168.300.100
```

---

### Schritt 4: Änderungen cluster-weit anwenden

**Web UI:** Datacenter → SDN → **Apply** *(wendet Konfiguration auf alle Nodes an)*

**Per CLI** (muss auf jedem Node ausgeführt werden):
```bash
# Auf allen Nodes anwenden
for node in pve1 pve2 pve3; do
  echo "=== $node ==="
  ssh root@$node "pvesh set /cluster/sdn && echo OK"
done

# Ergebnis prüfen: VNet-Bridges müssen vorhanden sein
ip link show type bridge | grep -E "prod-|mgmt-|dmz-|backup-"
```

Nach dem Apply ist jeder VNet-Name als Bridge auf allen Cluster-Nodes verfügbar und kann VMs zugewiesen werden.

---

## VMs in SDN VNets einbinden

### Neue VM anlegen

Im VM-Erstellungsdialog unter **Network → Bridge** den VNet-Namen wählen (z.B. `prod-v100`). Kein separates VLAN-Tag mehr nötig – das übernimmt der VNet.

```bash
# Per CLI
qm set <vmid> --net0 virtio,bridge=prod-v100
```

### Bestehende VM von vmbr1 auf VNet migrieren

```bash
# Aktuelle NIC-Konfiguration prüfen
qm config <vmid> | grep net

# Bridge tauschen (bei laufender VM → kurzer Verbindungsunterbruch)
qm set <vmid> --net0 virtio,bridge=prod-v100
```

> ⚠️ Bei laufenden VMs kurzen Verbindungsunterbruch einplanen. Für kritische VMs Wartungsfenster setzen.

---

## SDN & Firewall

Die `cluster.fw` aus Kapitel 07 schützt weiterhin den **Hypervisor** (Host-Traffic). VM-Traffic über SDN VNets wird zusätzlich auf VM-Ebene abgesichert.

### VM-Level-Firewall aktivieren

```bash
# Firewall für VM 100 einrichten
cat > /etc/pve/firewall/100.fw << 'EOF'
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: ACCEPT

[RULES]
IN ACCEPT -dport 22  -proto tcp  -comment "SSH"
IN ACCEPT -dport 80  -proto tcp  -comment "HTTP"
IN ACCEPT -dport 443 -proto tcp  -comment "HTTPS"
IN ACCEPT -proto icmp            -comment "ICMP"
EOF
```

### Inter-VNet-Isolation

Ohne Router zwischen VNets ist Inter-VLAN-Traffic automatisch blockiert (L2-Isolation durch getrennte VLANs). VMs in `prod-v100` können nicht direkt mit VMs in `mgmt-v200` kommunizieren – das muss explizit über einen Router/Firewall-VM erlaubt werden.

---

## Gesamtarchitektur nach SDN-Einführung

```
pve1 / pve2 / pve3  (identische SDN-Konfiguration nach Apply)
│
├── nic0 ──→ vmbr0 ─────────── 172.30.7.x/16        Management + Corosync Ring 1
├── ens19 ──────────────────── 10.10.20.x/24         Corosync Ring 0 (dediziert)
│
├── bond0 (active-backup)
│   ens20 + ens21
│   └──→ vmbr1 (Trunk-Bridge, VLAN-aware)
│         └── SDN Zone: vlan-zone
│               ├── VNet prod-v100   (VLAN 100) ──→ 192.168.100.0/24
│               ├── VNet mgmt-v200   (VLAN 200) ──→ 192.168.200.0/24
│               ├── VNet dmz-v300    (VLAN 300) ──→ 192.168.300.0/24
│               └── VNet backup-v400 (VLAN 400) ──→ 192.168.400.0/24
│
└── bond1 (balance-rr) ──────── 10.20.20.x/24        Ceph/Storage
    ens22 + ens23
```

---

## SDN-Konfigurationsdateien

| Datei | Inhalt | Repliziert via |
|-------|--------|----------------|
| `/etc/pve/sdn/zones.cfg` | Zone-Definitionen | pmxcfs ✓ |
| `/etc/pve/sdn/vnets.cfg` | VNet-Definitionen | pmxcfs ✓ |
| `/etc/pve/sdn/subnets.cfg` | Subnet/DHCP-Definitionen | pmxcfs ✓ |
| `/etc/network/interfaces.d/sdn` | Generierte Netzwerk-Config | lokal (pro Node) |

Die `.cfg`-Dateien werden durch pmxcfs cluster-weit synchronisiert. `interfaces.d/sdn` wird durch `pvesh set /cluster/sdn` aus der Cluster-Config generiert und ist node-lokal.

---

## Best Practices

✅ **Empfohlen:**
- VNets nach **Funktion** benennen (`prod-v100`, `mgmt-v200`) – nicht nach Kundennamen (können sich ändern)
- VLAN-Tags mit Switch-Konfiguration abgleichen bevor Apply (erlaubte VLANs am Trunk-Port)
- Subnets für alle VNets definieren – auch ohne DHCP, rein für Dokumentationszwecke
- SDN-Apply zuerst auf einem Node testen, dann cluster-weit ausrollen
- Änderungen an `/etc/pve/sdn/*.cfg` via Git versionieren (Backup der pmxcfs-Dateien)

⚠️ **Achtung:**
- `pvesh set /cluster/sdn` muss nach jeder Änderung auf **jedem Node** ausgeführt werden (oder Web UI Apply)
- VNets löschen: Erst alle VMs aus dem VNet entfernen, dann VNet löschen
- VLAN-Tags am Switch-Trunk-Port als erlaubte VLANs eintragen, sonst kein Traffic
- Bestehende VMs auf `vmbr1` nach SDN-Einführung **auf VNets migrieren** – `vmbr1` bleibt weiter als Trunk-Bridge, sollte aber nicht mehr direkt für VMs genutzt werden

❌ **Vermeiden:**
- Gleiche VLAN-IDs in verschiedenen Zonen (führt zu Konflikten)
- SDN für Management-, Corosync- oder Ceph-Netzwerke verwenden
- VMs direkt auf `vmbr1` lassen wenn SDN aktiv – Verwaltbarkeit leidet

---

## Häufige Fehler & Lösungen

| Problem | Mögliche Ursache | Lösung |
|---------|-----------------|--------|
| VNet-Bridge fehlt auf einem Node | `pvesh set /cluster/sdn` nicht ausgeführt | SSH auf Node → `pvesh set /cluster/sdn` |
| VM hat nach VNet-Wechsel keinen Traffic | VLAN am Switch nicht erlaubt | Switch-Trunk-Config prüfen (erlaubte VLANs) |
| DHCP-Vergabe funktioniert nicht | dnsmasq fehlt oder läuft nicht | `apt install dnsmasq`; `systemctl list-units | grep dnsmasq` |
| VNet erscheint nicht im Bridge-Dropdown | SDN Apply nicht ausgeführt | Web UI: Datacenter → SDN → Apply |
| `pvesh create /cluster/sdn/zones` schlägt fehl | Package fehlt | `apt install libpve-network-perl` |
| Inter-VNet-Traffic unerwartet möglich | Router-VM zwischen VNets vorhanden | Routing auf Router-VM einschränken / VM-Firewall aktivieren |

### Debugging-Befehle

```bash
# SDN-Konfiguration anzeigen
pvesh get /cluster/sdn/zones
pvesh get /cluster/sdn/vnets

# Generierte Bridges auf dem Node prüfen
ip link show type bridge
bridge link show

# VLAN-Zuordnung der Bridge-Ports prüfen
bridge vlan show

# SDN-Netzwerk-Config anzeigen (generierte Datei)
cat /etc/network/interfaces.d/sdn

# dnsmasq-Instanzen für SDN-Subnets
systemctl list-units | grep dnsmasq

# Logs bei Apply-Problemen
journalctl -u pvenetcommit --since "5 minutes ago"
```

---

## Weiterführende Links

- [Proxmox SDN – Offizielle Dokumentation](https://pve.proxmox.com/wiki/SDN)
- [Proxmox SDN Konfigurationsreferenz](https://pve.proxmox.com/pve-docs/chapter-pvesdn.html)
- [Kapitel 02 – Netzwerk & Clustering](02-netzwerk-clustering.md)
- [Kapitel 07 – Firewall & IPsets](07-firewall-ipsets.md)
