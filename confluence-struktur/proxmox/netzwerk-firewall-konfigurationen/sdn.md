<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: Proxmox - Netzwerk- & Firewall-Konfigurationen -->
<!-- Title: Proxmox - SDN -->
<!-- Label: pve-netzwerk, pve-cluster, best-practice, referenz -->

# Software Defined Networking (SDN)

> **PVE 9.x (Debian 13 Trixie)** · Letzte Aktualisierung: 2026-06
> Quelle: adaptiert aus Kapitel 08 – SDN.
> Verwandte Seiten: [Ceph Cluster - Netzwerk](ceph-cluster-netzwerk.md) · [Firewall](firewall.md)

---

## Wann anwenden?

- **Immer beim Cluster-Betrieb** – SDN ist die Best-Practice-Methode für VM-Netzwerke in Proxmox-Clustern
- **Multi-Tenant-Umgebungen** – Trennung von Kunden-/Projektnetzen über VNets statt manueller Bridge-Configs
- **VLAN-Management vereinfachen** – einmal anlegen, automatisch auf alle Nodes propagiert

> SDN gilt ausschließlich für VM-Traffic (bond0 / vmbr1). Corosync, Ceph und Management bleiben physisch dediziert und werden nicht über SDN verwaltet.

---

## Überblick

Ohne SDN muss jede neue VM-VLAN auf **jedem Cluster-Node einzeln** als Linux-Bridge konfiguriert werden. SDN löst das: VNets werden einmal im Datacenter angelegt und automatisch auf alle Nodes propagiert.

```
Ohne SDN: 3× Bridge-Konfiguration pro neuer VLAN
Mit SDN:  1× VNet anlegen → Apply → auf allen Nodes verfügbar
```

### Hyper-V-Vergleich

| Konzept | Hyper-V | Proxmox SDN |
|---|---|---|
| VM-Netzwerk | Virtuelle Switches (pro Host) | VNets (cluster-weit, zentral) |
| DHCP-Server | Separate Windows-DHCP-Rolle | Optional direkt in SDN via dnsmasq |
| Zentrale Verwaltung | SCVMM / Windows Admin Center | Datacenter → SDN (nativ) |

### SDN-Zonentypen im Vergleich

| Zonentyp | Einsatzgebiet | Komplexität |
|---|---|---|
| VLAN | Standard-VLAN, Single-Site | ⭐ Niedrig |
| QinQ | VLAN-Stacking, Provider-Netze | ⭐⭐ Mittel |
| VXLAN | L2-Overlay, Multi-Site | ⭐⭐ Mittel |
| EVPN | Verteiltes L3-Routing mit BGP | ⭐⭐⭐ Hoch |

> **VIREQ-Standard: VLAN Zone** – passt zu den meisten Kunden-Installationen, gleiche Switches/VLANs wie bisher.

---

## Voraussetzungen

```bash
dpkg -l libpve-network-perl | grep ^ii
apt update && apt install libpve-network-perl   # falls nicht vorhanden
```

`vmbr1` muss bereits als VLAN-aware Bridge auf `bond0` konfiguriert sein.

---

## Schritt 1: Zone erstellen

Auch über **Web UI: Datacenter → SDN → Zones → Add → VLAN** möglich.

> 📸 **Screenshot machen:** Datacenter → SDN → Zones → Add (VLAN)-Dialog mit ID `vlan-zone` und Bridge `vmbr1` ausgefüllt – das ist der erste Schritt, den jeder neue Techniker bei euch sehen sollte, bevor er VNets anlegt.

```bash
pvesh create /cluster/sdn/zones --zone vlan-zone --type vlan --bridge vmbr1
```

## Schritt 2: VNets erstellen

| VNet-Name | VLAN-Tag | Verwendung |
|---|---|---|
| `prod-v100` | 100 | Produktivnetz |
| `mgmt-v200` | 200 | Management |
| `dmz-v300` | 300 | DMZ |

> 📸 **Screenshot machen:** Datacenter → SDN → VNets → Übersichtsliste mit allen vier VNets (prod/mgmt/dmz/backup) inklusive zugeordneter Zone und VLAN-Tag-Spalte – zeigt auf einen Blick, wie die VLAN-zu-VNet-Zuordnung in der Praxis aussieht.

```bash
pvesh create /cluster/sdn/vnets --vnet prod-v100 --zone vlan-zone --tag 100
pvesh create /cluster/sdn/vnets --vnet mgmt-v200 --zone vlan-zone --tag 200
pvesh create /cluster/sdn/vnets --vnet dmz-v300  --zone vlan-zone --tag 300
```

## Schritt 3: Subnets & DHCP (optional)

```bash
for node in pve1 pve2 pve3; do ssh root@$node "apt install -y dnsmasq"; done

pvesh create /cluster/sdn/vnets/prod-v100/subnets \
  --subnet 192.168.100.0/24 --type subnet --gateway 192.168.100.1 \
  --dhcp-range start-address=192.168.100.100,end-address=192.168.100.200
```

## Schritt 4: Cluster-weit anwenden

```bash
# Web UI: Datacenter → SDN → Apply
for node in pve1 pve2 pve3; do ssh root@$node "pvesh set /cluster/sdn"; done
ip link show type bridge | grep -E "prod-|mgmt-|dmz-"
```

> 📸 **Screenshot machen:** Datacenter → SDN → die Hauptübersichtsseite mit dem grünen "Apply"-Button und der Liste der "Pending Changes" davor – das ist der Moment, in dem viele Anfänger vergessen, dass Änderungen erst nach Apply wirksam werden.

---

## VMs in SDN VNets einbinden

```bash
qm set <vmid> --net0 virtio,bridge=prod-v100
```

> ⚠️ Bei laufenden VMs kurzen Verbindungsunterbruch einplanen.

---

## SDN & Firewall

Die `cluster.fw` schützt weiterhin den Hypervisor (→ [Firewall](firewall.md)). VM-Traffic über SDN VNets wird zusätzlich auf VM-Ebene per `/etc/pve/firewall/<vmid>.fw` abgesichert. Ohne Router zwischen VNets ist Inter-VLAN-Traffic automatisch blockiert.

---

## SDN-Konfigurationsdateien

| Datei | Inhalt | Repliziert via |
|---|---|---|
| `/etc/pve/sdn/zones.cfg` | Zone-Definitionen | pmxcfs ✓ |
| `/etc/pve/sdn/vnets.cfg` | VNet-Definitionen | pmxcfs ✓ |
| `/etc/network/interfaces.d/sdn` | Generierte Netzwerk-Config | lokal (pro Node) |

---

## Best Practices

✅ VNets nach Funktion benennen (`prod-v100`), nicht nach Kundennamen
✅ VLAN-Tags mit Switch-Konfiguration abgleichen, bevor Apply
✅ SDN-Apply zuerst auf einem Node testen, dann cluster-weit ausrollen

⚠️ `pvesh set /cluster/sdn` muss nach jeder Änderung auf jedem Node ausgeführt werden
⚠️ VLAN-Tags am Switch-Trunk-Port als erlaubte VLANs eintragen, sonst kein Traffic

❌ Gleiche VLAN-IDs in verschiedenen Zonen
❌ SDN für Management-, Corosync- oder Ceph-Netzwerke verwenden

---

## Häufige Fehler & Lösungen

| Problem | Ursache | Lösung |
|---|---|---|
| VNet-Bridge fehlt auf einem Node | `pvesh set /cluster/sdn` nicht ausgeführt | SSH → `pvesh set /cluster/sdn` |
| VM hat nach VNet-Wechsel keinen Traffic | VLAN am Switch nicht erlaubt | Switch-Trunk-Config prüfen |
| DHCP funktioniert nicht | dnsmasq fehlt | `apt install dnsmasq` |

---

## Weiterführende Links

| Ressource | URL |
|---|---|
| Proxmox SDN | https://pve.proxmox.com/wiki/SDN |
| SDN Konfigurationsreferenz | https://pve.proxmox.com/pve-docs/chapter-pvesdn.html |
