<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: Proxmox - Netzwerk- & Firewall-Konfigurationen -->
<!-- Title: Proxmox - Ceph Cluster - Netzwerk -->
<!-- Label: pve-netzwerk, pve-storage, pve-cluster, best-practice, checkliste -->

# Ceph Cluster – Netzwerk

> **PVE 9.x / Ceph 19.x Squid (Debian 13 Trixie)** · Letzte Aktualisierung: 2026-06
> Referenz-Topologie: 3-Node-Cluster `pve1`/`pve2`/`pve3`, 6 NICs pro Node (siehe Kapitel 07, NIC-Layout)
> Verwandte Seiten: [Firewall](firewall.md) · [Port-Referenz](port-referenz.md)

---

## Wann anwenden?

- Ab **3 Nodes** mit Ceph als Storage-Backend – das ist der VIREQ-Standard für hyper-converged Cluster
- Voraussetzung für Live-Migration und HA-Manager (shared Storage erforderlich)

> **Nicht anwenden** für 2-Node-Cluster – Ceph braucht ein eigenes Monitor-Quorum und mindestens 3 Nodes. Für 2 Nodes: NFS oder PBS als Storage, siehe [2 Node Cluster + QDevice - Netzwerk](2-node-cluster-qdevice-netzwerk.md).

---

## Überblick

Ceph braucht **vier** logisch getrennte Verkehrsarten, die im Referenz-Setup auf physisch getrennte NICs/Bonds verteilt werden, damit sich Storage-Traffic, Cluster-Heartbeat und VM-Traffic nicht gegenseitig beeinflussen:

| Verkehrsart | Netz im Referenz-Setup | Interface |
|---|---|---|
| Management / Web-UI / Admin-SSH | 172.30.7.0/16 | `nic0` → `vmbr0` |
| Corosync Ring 1 (Fallback) | 172.30.7.0/16 | `vmbr0` (mitgenutzt) |
| Corosync Ring 0 (primär) | 10.10.20.0/24 | `ens19` (dediziert) |
| VM-Traffic (Kunden-VLANs) | – (VLAN-getaggt) | `bond0` (active-backup) → `vmbr1` |
| Ceph Public + Cluster Network | 10.20.20.0/24 | `bond1` (balance-rr) |

> **Produktiv-Empfehlung:** Ceph Public Network (Client-I/O) und Cluster Network (OSD-Replikation) auf getrennten Subnetzen, wenn genügend NICs vorhanden sind. Im VIREQ-Referenz-Setup teilen sich beide ein kombiniertes Netz (`10.20.20.0/24` auf `bond1`) – für die meisten SMB-Deployments ausreichend, bei sehr I/O-intensiven Workloads als Erweiterungsoption prüfen.

---

## Voraussetzungen

- Mindestens 3 Nodes mit jeweils 4–6 NICs (volle Trennung von Mgmt/Corosync/VM/Ceph)
- [Post Installation Check](../installationen/post-installation-check.md) auf allen Nodes abgeschlossen
- Mindestens 1 dedizierte Disk pro OSD pro Node (keine OS-Disk!)
- Hardware-RAID-Controller im HBA-IT-Mode (→ Kapitel 01, Abschnitt 1.2) – Ceph braucht direkten Disk-Zugriff

---

## Schritt für Schritt

### 1. IP-Schema festlegen (physisch getrennt, ohne Überlappung)

```
172.30.0.0/16    → Management + Corosync Ring 1
10.10.20.0/24    → Corosync Ring 0
10.20.20.0/24    → Ceph Public + Cluster Network
(VLAN-getaggt)   → VM-Traffic
```

### 2. NIC-Zuordnung und Bonds einrichten

`/etc/network/interfaces` auf `pve1` (analog auf `pve2`/`pve3`, nur letztes Oktett der IPs anpassen):

```
auto lo
iface lo inet loopback

# Management + Corosync Ring 1
auto nic0
iface nic0 inet manual

auto vmbr0
iface vmbr0 inet static
    address 172.30.7.31/16
    gateway 172.30.254.254
    bridge-ports nic0
    bridge-stp off
    bridge-fd 0

# Corosync Ring 0 – dedizierte NIC, kein Bonding (Corosync regelt Redundanz selbst)
auto ens19
iface ens19 inet static
    address 10.10.20.11/24

# VM-Traffic – bond0 active-backup, VLAN-aware
auto ens20
iface ens20 inet manual

auto ens21
iface ens21 inet manual

auto bond0
iface bond0 inet manual
    bond-slaves ens20 ens21
    bond-miimon 100
    bond-mode active-backup
    bond-primary ens20

auto vmbr1
iface vmbr1 inet manual
    bridge-ports bond0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094

# Ceph-Netz – bond1 balance-rr, physisch isoliert, kein Gateway
auto ens22
iface ens22 inet manual

auto ens23
iface ens23 inet manual

auto bond1
iface bond1 inet manual
    bond-slaves ens22 ens23
    bond-miimon 100
    bond-mode balance-rr

auto vmbr2
iface vmbr2 inet static
    address 10.20.20.11/24
    bridge-ports bond1
    bridge-stp off
    bridge-fd 0

dns-nameservers 172.30.254.254
```

> **`balance-rr` auf bond1:** Round-Robin verteilt Pakete sequenziell über beide NICs – dafür ist Switch-seitige Konfiguration nötig (vergleichbar mit LACP, aber ohne 802.3ad-Aushandlung). Mit dem Kunden-Netzwerkteam vorab abstimmen, dass die beiden Switch-Ports für Round-Robin korrekt konfiguriert sind – sonst kann es zu Out-of-Order-Paketen und TCP-Retransmits kommen.
> **Warum kein Bonding auf der Corosync-NIC?** Corosync kann selbst zwischen mehreren konfigurierten Ringen (Ring 0/Ring 1) umschalten. Ein zusätzlicher Bond würde nur Komplexität ohne Mehrwert bringen – manche Bond-Modi gelten zudem als problematisch für Corosync.

```bash
ifreload -a
ping -c5 10.20.20.12   # Ceph-Netz-Konnektivität zu pve2 prüfen
```

### 3. Optional: Jumbo Frames (MTU 9000) auf dem Ceph-Netz

Bei größeren sequenziellen Storage-Transfers reduziert eine höhere MTU den Paket-Overhead spürbar. **MTU muss durchgängig** auf NIC, Bond, Bridge und Switch-Port gleich gesetzt sein – inkonsistente MTU führt zu Fragmentierung oder stillem Paketverlust:

```
# Ergänzung in /etc/network/interfaces für ens22, ens23, bond1, vmbr2:
    mtu 9000
```

```bash
# Nach ifreload -a prüfen:
ip link show bond1 | grep mtu
ping -M do -s 8972 10.20.20.12   # Test ohne Fragmentierung bei MTU 9000
```

> Vor dem produktiven Einsatz mit dem Kunden-Netzwerkteam abstimmen, dass die Switch-Ports auf dem Ceph-Netz ebenfalls Jumbo Frames erlauben.

### 4. Cluster erstellen und alle 3 Nodes beitreten lassen

```bash
# Auf pve1:
pvecm create CEPHCLUSTER \
    --link0 address=10.10.20.11 \
    --link1 address=172.30.7.31

# Auf pve2 und pve3:
pvecm add 172.30.7.31 \
    --link0 address=10.10.20.1X \
    --link1 address=172.30.7.3X

pvecm status   # Alle 3 Nodes, Quorate: Yes
```

### 5. Ceph initialisieren – explizit auf das Ceph-Netz zeigen

```bash
# Auf pve1:
pveceph init --network 10.20.20.0/24

cat /etc/ceph/ceph.conf
# public network = 10.20.20.0/24
# cluster network = 10.20.20.0/24
```

> Kritischer Punkt: Wird hier nicht explizit das Ceph-Netz angegeben, initialisiert Ceph standardmäßig auf dem Management-Netz – das läuft dann zwar, belastet aber das falsche Netz und widerspricht der physischen Trennung.

### 6. Monitore, Manager und OSDs einrichten

```bash
# Monitor je Node
pveceph mon create
ssh root@pve2 "pveceph mon create"
ssh root@pve3 "pveceph mon create"
ceph mon stat   # quorum 0,1,2 pve1,pve2,pve3

# Manager (aktiv + standby)
pveceph mgr create
ssh root@pve2 "pveceph mgr create"

# OSDs (Details und Pool-Konfiguration → Kapitel 09 – Ceph Storage)
```

### 7. Firewall: Ceph-Netz freigeben

Das `ceph-net`-IPset nutzt Subnetz-Notation (`/24`), weil `bond1` physisch isoliert ist und kein Gateway hat – vollständige Regeln → [Firewall](firewall.md), Ports → [Port-Referenz](port-referenz.md):

```ini
[IPSET ceph-net]
10.20.20.0/24

[RULES]
IN ACCEPT -source +dc/ceph-net -p tcp -dport 3300      -log nolog # Ceph Monitor v2
IN ACCEPT -source +dc/ceph-net -p tcp -dport 6789      -log nolog # Ceph Monitor v1
IN ACCEPT -source +dc/ceph-net -p tcp -dport 6800:7300 -log nolog # Ceph OSD/MDS/MGR
IN ACCEPT -source +dc/ceph-net -p icmp                 -log nolog # ICMP: Ceph-Netz
```

### 8. Funktionstest

```bash
ceph -s
ceph osd tree

# OSD-Heartbeat-Latenz im Ceph-Netz
ceph osd perf

# Failover-Test: einen Node trennen, Cluster-Reaktion beobachten
ceph health detail   # erwartet: HEALTH_WARN während Re-Replikation, danach HEALTH_OK
```

### 9. Dokumentation

NIC-Zuordnung, IP-Schema, Bond-Modi und ggf. MTU-Einstellung im Confluence-Eintrag des Clusters festhalten – inklusive der Begründung, warum Public/Cluster Network kombiniert statt getrennt laufen (falls so entschieden).

---

## Best Practices

✅ Ceph-Netz physisch isoliert ohne Gateway – erlaubt Subnetz-Notation im IPset statt Einzeladressen
✅ `balance-rr` nur mit vorab abgestimmter Switch-Konfiguration einsetzen
✅ Ceph-Init immer mit explizitem `--network`-Parameter, nie auf den Default verlassen
✅ Bei ausreichend NICs: Public und Cluster Network trennen, sobald I/O-Last es rechtfertigt
✅ Jumbo Frames nur durchgängig (NIC, Bond, Bridge, Switch) oder gar nicht einsetzen

⚠️ Hardware-RAID-Controller sind mit Ceph inkompatibel – HBA-IT-Mode zwingend
⚠️ Bei `HEALTH_WARN` nie voreilig OSDs entfernen – Rebalancing abwarten

❌ Public/Cluster Network über das Management-Interface laufen lassen
❌ Ceph auf weniger als 3 Nodes betreiben
❌ OS-Disks als OSD verwenden

---

## Häufige Fehler & Lösungen

**MON nicht im Quorum**
```
Ursache: Ceph-Netz nicht erreichbar.
Prüfen:  ceph mon stat; bond1-Konnektivität (ping 10.20.20.12)
```

**`HEALTH_WARN: slow ops`**
```
Ursache: HDD-Latenz oder Netzwerk-Engpass auf bond1.
Prüfen:  ceph osd perf; Bond1-Traffic und MTU-Konsistenz prüfen
```

**Ceph initialisiert auf dem falschen Netz**
```
Ursache: pveceph init ohne --network ausgeführt.
Lösung:  /etc/ceph/ceph.conf manuell auf 10.20.20.0/24 korrigieren, Dienste neu starten
```

---

## Weiterführende Links

| Ressource | URL |
|---|---|
| Proxmox Ceph – Hyper-Converged Setup | https://pve.proxmox.com/wiki/Deploy_Hyper-Converged_Ceph_Cluster |
| Ceph Network Configuration Reference | https://docs.ceph.com/en/squid/rados/configuration/network-config-ref/ |
