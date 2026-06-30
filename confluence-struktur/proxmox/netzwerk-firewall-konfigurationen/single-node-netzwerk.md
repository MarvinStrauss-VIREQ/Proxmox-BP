<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: Proxmox - Netzwerk- & Firewall-Konfigurationen -->
<!-- Title: Proxmox - Single Node - Netzwerk -->
<!-- Label: pve-netzwerk, best-practice, checkliste -->

# Single Node – Netzwerk

> **PVE 9.x (Debian 13 Trixie)** · Letzte Aktualisierung: 2026-06
> Verwandte Seiten: [PVE Erstinstallation](../installationen/pve-erstinstallation.md) · [Firewall](firewall.md) · [Port-Referenz](port-referenz.md)

---

## Wann anwenden?

- Einzelner PVE-Host **ohne** Cluster, ohne HA, ohne Ceph
- Typisch für: kleine Standorte, Außenstellen, Testlabs, Einzelserver-Kundenprojekte
- Migration zu einem Cluster ist später jederzeit möglich – dieses Netzwerk-Schema ist dafür bereits kompatibel

> **Nicht anwenden**, wenn von vornherein klar ist, dass ein zweiter Node oder Ceph dazukommt – dann direkt mit [2-Node + QDevice](2-node-cluster-qdevice-netzwerk.md) oder [Ceph Cluster](ceph-cluster-netzwerk.md) planen, das spart eine spätere Migration der Netzwerkkonfiguration.

---

## Überblick

Beim Single Node gibt es keinen Corosync-Traffic und kein Storage-Netz im Sinne von Ceph – die einzige relevante Trennung ist **Management vs. VM-Traffic**. Wie viele Bridges/Bonds sinnvoll sind, hängt direkt von der NIC-Anzahl ab:

| NIC-Anzahl | Empfohlenes Schema |
|---|---|
| 1 NIC | `vmbr0` trägt Management **und** VM-Traffic gemeinsam |
| 2 NICs (kein Redundanzbedarf) | `vmbr0` (Management) + `vmbr1` (VM-Traffic, VLAN-aware) |
| 2 NICs (Redundanzbedarf) | `bond0` (active-backup) aus beiden NICs → `vmbr0` für Management + VM-Traffic kombiniert |
| 4 NICs | `bond0` (Management) + `bond1` (VM-Traffic) – volle Redundanz auf beiden Ebenen |

> **Hyper-V-Vergleich:** Bei einem Single-Hyper-V-Host ohne Failover-Cluster ist das Äquivalent ein einzelner virtueller Switch über NIC-Teaming. Proxmox bildet das 1:1 mit `vmbr0` über einem `active-backup`-Bond ab – ohne Switch-seitige Konfiguration nötig.

---

## Voraussetzungen

- [PVE Erstinstallation](../installationen/pve-erstinstallation.md) abgeschlossen, Web-UI erreichbar
- IP-Schema für Management-Netz definiert
- Bei VLANs für Kunden-VMs: VLAN-IDs vom Kunden-Netzwerkteam bekannt

---

## Schritt für Schritt

### 1. NIC-Inventur

```bash
ip link show
ethtool eno1 | grep -i speed
lspci -nn | grep -i ethernet
```

Intel-NICs (`igb`, `ixgbe`, `i40e`) sind die sichere Wahl (→ Kapitel 01, NIC-Kompatibilität). Broadcom vorab validieren.

### 2. IP-Schema festlegen

Beispiel für einen Single-Node-Standort:

```
192.168.10.0/24   → Management (Web-UI, SSH, API)
192.168.20.0/24   → VM-Traffic (ggf. mit mehreren VLANs darunter)
```

### 3a. Schema mit nur einer NIC

`/etc/network/interfaces`:

```
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.10.10/24
    gateway 192.168.10.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094

dns-nameservers 192.168.10.1
```

> VM-Traffic läuft hier auf demselben physischen Link wie das Management – für Kunden-VLANs auf `vmbr0` einfach beim VM-Anlegen das Tag mitgeben (VLAN-aware Bridge macht den Rest).

### 3b. Schema mit zwei NICs, ohne Redundanzbedarf

```
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual

auto eno2
iface eno2 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.10.10/24
    gateway 192.168.10.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0

auto vmbr1
iface vmbr1 inet manual
    bridge-ports eno2
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094

dns-nameservers 192.168.10.1
```

`vmbr1` bekommt bewusst **keine IP** – sie transportiert nur VM-Traffic, das Hostsystem selbst braucht hier keine Adresse.

### 3c. Schema mit zwei NICs und Redundanz (VIREQ-Standard)

```
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual

auto eno2
iface eno2 inet manual

auto bond0
iface bond0 inet manual
    bond-slaves eno1 eno2
    bond-miimon 100
    bond-mode active-backup
    bond-primary eno1

auto vmbr0
iface vmbr0 inet static
    address 192.168.10.10/24
    gateway 192.168.10.1
    bridge-ports bond0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094

dns-nameservers 192.168.10.1
```

> VIREQ-Standard ist `active-backup`, nicht LACP – funktioniert ohne Switch-Konfiguration und ist damit auch bei Kunden mit unmanaged Switches einsetzbar.

Konfiguration anwenden (kein Reboot nötig dank `ifupdown2`):

```bash
ifreload -a
ip addr show vmbr0
```

### 4. Hostname & DNS prüfen

```bash
hostname --fqdn
cat /etc/hosts
# Hostname MUSS auf die Management-IP zeigen, nie auf 127.0.x.x
```

### 5. Firewall-Basis aktivieren

Auch im Single-Node-Fall: Admin-IP eintragen, dann aktivieren (vollständige Anleitung → [Firewall](firewall.md)):

```bash
nano /etc/pve/firewall/cluster.fw
pve-firewall compile
pvesh set /cluster/firewall/options --enable 1
pve-firewall status
```

### 6. Funktionstest

```bash
# Web-UI erreichbar?
curl -k -o /dev/null -s -w "%{http_code}\n" https://192.168.10.10:8006

# Bei Bond: Failover testen – aktives Kabel ziehen, dann:
watch -n1 cat /proc/net/bonding/bond0
```

Erwartung im Bond-Status: nach max. 100–200 ms (`bond-miimon`) wechselt `MII Status` der Slaves, `vmbr0` bleibt durchgängig erreichbar (Ping-Test parallel laufen lassen).

### 7. Dokumentation

IP-Schema, NIC-Zuordnung, Bond-Modus und Firewall-Status im Confluence-Eintrag des Standorts/Kunden festhalten.

---

## Best Practices

✅ `vmbr0` für Management auch bei VM-Traffic-Mitnutzung VLAN-aware anlegen – kostet nichts, erlaubt aber spätere Segmentierung ohne Neukonfiguration
✅ `active-backup` als Standard-Bond-Modus, kein LACP ohne abgestimmte Switch-Konfiguration
✅ Firewall-Basis auch im Single-Node-Fall aktivieren – ein ungeschützter Host bleibt ein ungeschützter Host
✅ IP-Schema so wählen, dass eine spätere Cluster-Erweiterung (Corosync-Netz) ohne Subnetz-Konflikt möglich ist

⚠️ Bei nur einer NIC: kein Netzwerk-Ausfallschutz für den gesamten Host – das im Kundengespräch transparent machen
⚠️ Netzwerkänderungen am Single Node immer mit iDRAC/iLO-Fallback bereithalten

❌ VM-Traffic und Management dauerhaft ungetrennt lassen, wenn Kunden-VLANs im Spiel sind
❌ Realtek-NICs für produktiven VM-Traffic einsetzen

---

## Häufige Fehler & Lösungen

**Web-UI nach `ifreload -a` nicht mehr erreichbar**
```
Ursache: Tippfehler in Bridge-Konfiguration oder falsches Gateway.
Lösung:  Über iDRAC/iLO-Konsole /etc/network/interfaces korrigieren, erneut ifreload -a.
```

**Bond zeigt `MII Status: down` für beide Slaves**
```
Ursache: Kabel/Switch-Port-Problem oder falscher bond-mode für die Verkabelung.
Lösung:  ethtool <nic> prüfen, Verkabelung kontrollieren.
```

---

## Weiterführende Links

| Ressource | URL |
|---|---|
| PVE Network Configuration | https://pve.proxmox.com/wiki/Network_Configuration |
| Linux Bonding | https://pve.proxmox.com/pve-docs/pve-admin-guide.html#sysadmin_network_bond |
