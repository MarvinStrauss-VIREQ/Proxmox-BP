<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: Proxmox - Netzwerk- & Firewall-Konfigurationen -->
<!-- Title: Proxmox - backup Server - Netzwerk -->
<!-- Label: pve-netzwerk, best-practice, checkliste -->

# Backup Server – Netzwerk

> **PBS 4.x (Debian 13 Trixie)** · Letzte Aktualisierung: 2026-06
> Verwandte Seiten: [Backup Server Erstinstallation](../installationen/backup-server-erstinstallation.md) · [2 Node Cluster + QDevice - Netzwerk](2-node-cluster-qdevice-netzwerk.md) · [Firewall](firewall.md) · [Port-Referenz](port-referenz.md)

---

## Wann anwenden?

- Bei **jeder** PBS-Installation – laut VIREQ-Standard wird PBS immer mitgeplant, egal ob Single-Node, 2-Node+QDevice oder Ceph-Cluster
- Wenn PBS zusätzlich als QDevice für einen 2-Node-Cluster dient (→ [2 Node Cluster + QDevice - Netzwerk](2-node-cluster-qdevice-netzwerk.md))

---

## Überblick

PBS braucht im einfachsten Fall nur eine Management-IP im selben Netz wie die PVE-Nodes. Ein eigenes Backup-VLAN wird erst relevant, wenn Backup-Traffic das Produktionsnetz spürbar belastet – typischerweise bei großen VM-Beständen mit täglichen Full-Backups.

| Szenario | Empfehlung |
|---|---|
| Kleine Umgebung (wenige VMs, kleine Datenmengen) | PBS im Management-Netz, kein eigenes VLAN nötig |
| Große VM-Bestände / tägliche Fulls | Eigenes Backup-VLAN, um Produktionsnetz nicht zu sättigen |
| PBS dient gleichzeitig als QDevice | PBS muss aus dem Corosync-relevanten Netz erreichbar sein – Management-Netz meist ausreichend |

> **Hyper-V-Vergleich:** PBS ≈ eine Backup-VM auf NAS-Storage – mit dem Unterschied, dass das Netzwerk-Setup hier identisch zum Setup eines normalen PVE-Nodes ist (gleiche Bonding-/Bridge-Logik), nicht ein separates NAS-Protokoll.

---

## Voraussetzungen

- [Backup Server Erstinstallation](../installationen/backup-server-erstinstallation.md) abgeschlossen
- IP-Schema der PVE-Nodes bekannt (PBS muss von allen Nodes erreichbar sein)
- Bei QDevice-Doppelrolle: Corosync-Netz-IPs der PVE-Nodes bekannt

---

## Schritt für Schritt

### 1. IP-Schema festlegen

```
192.168.1.0/24   → Management (PBS-IP im selben Netz wie die PVE-Nodes)
192.168.3.0/24   → optional: dediziertes Backup-VLAN, falls Traffic-Volumen es erfordert
```

### 2. Prüfen, ob ein eigenes Backup-Netz nötig ist

Faustregel: Wenn die tägliche Backup-Fenster-Dauer das Produktionsnetz während Geschäftszeiten belastet (z. B. Replikation/Migration gleichzeitig läuft), ein eigenes VLAN vorsehen. Sonst reicht das Management-Netz.

### 3a. Einfaches Schema (eine NIC, Management-Netz)

`/etc/network/interfaces` auf dem PBS-Server:

```
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.1.50/24
    gateway 192.168.1.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0

dns-nameservers 192.168.1.1
```

### 3b. Mit dediziertem Backup-VLAN (zwei NICs, Redundanz via Bond)

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

# Management
auto vmbr0
iface vmbr0 inet static
    address 192.168.1.50/24
    gateway 192.168.1.1
    bridge-ports bond0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094

# Dediziertes Backup-VLAN (Subinterface auf der VLAN-aware Bridge)
auto vmbr0.30
iface vmbr0.30 inet static
    address 192.168.3.50/24

dns-nameservers 192.168.1.1
```

> Wichtig: Wenn PBS über das Backup-VLAN angesprochen werden soll, müssen auch die PVE-Nodes ein Interface in diesem VLAN haben – sonst läuft der Backup-Traffic trotzdem über das Management-Netz, weil das die einzige gemeinsame Route ist.

```bash
ifreload -a
```

### 4. Firewall: Datastore-Zugriff einschränken

PBS sollte nur von den PVE-Node-IPs erreichbar sein (analog zum `mgmt-hosts`-Pattern aus Kapitel 07). Vollständige Regeln → [Firewall](firewall.md), Ports → [Port-Referenz](port-referenz.md):

```ini
# /etc/proxmox-backup/firewall.cfg ähnliches Prinzip – bzw. iptables/nftables auf PBS direkt,
# da PBS keine eigene pve-firewall-Instanz wie PVE hat.

# Beispiel mit nftables:
nft add rule inet filter input ip saddr 192.168.1.10 tcp dport 8007 accept  # pve01
nft add rule inet filter input ip saddr 192.168.1.11 tcp dport 8007 accept  # pve02
nft add rule inet filter input ip saddr 192.168.1.50 tcp dport 8007 accept  # PBS selbst (Web-UI)
```

> PBS bringt keine eigene `pve-firewall`-Instanz mit – Netzwerk-Einschränkungen laufen über Standard-Linux-Firewalling (nftables/iptables) oder über die vorgeschaltete Netzwerksegmentierung (VLAN/ACL am Switch).

### 5. Bei QDevice-Doppelrolle: zusätzlichen Port freigeben

Falls dieser PBS-Server auch als QDevice für einen 2-Node-Cluster dient, zusätzlich TCP 5403 für die beiden PVE-Nodes freigeben (→ [2 Node Cluster + QDevice - Netzwerk](2-node-cluster-qdevice-netzwerk.md), Schritt 5–6).

### 6. Bei Offsite-Sync zu einer zweiten PBS-Instanz

```bash
# Sync-Job läuft über HTTPS (Port 8007) – Ziel-IP/Netz in der Firewall freigeben
nft add rule inet filter output ip daddr <ZIEL-PBS-IP> tcp dport 8007 accept
```

### 7. Funktionstest

```bash
# Datastore von allen PVE-Nodes aus anbinden
# PVE Web-UI: Datacenter → Storage → Add → Proxmox Backup Server

# Erreichbarkeit prüfen:
openssl s_client -connect 192.168.1.50:8007 | openssl x509 -fingerprint -noout

# Testbackup einer VM auslösen und Datastore-Auslastung prüfen
proxmox-backup-manager datastore list
```

### 8. Dokumentation

IP, Datastore-Pfad, ggf. Backup-VLAN-ID und Sync-Ziel (falls Offsite) dokumentieren.

---

## Best Practices

✅ PBS auf dedizierter Hardware, nicht als VM auf dem gesicherten Cluster selbst
✅ Eigenes Backup-VLAN erst einplanen, wenn das Traffic-Volumen es rechtfertigt – sonst unnötige Komplexität
✅ Bei QDevice-Doppelrolle: PBS muss aus dem Corosync-relevanten Netz erreichbar sein, nicht nur aus dem Backup-VLAN
✅ 3-2-1-Regel auch netzwerktechnisch denken: Offsite-Sync-Ziel auf einer getrennten Route/VPN-Verbindung

⚠️ Wird PBS über ein eigenes VLAN angesprochen, müssen auch die PVE-Nodes ein Interface in diesem VLAN haben
⚠️ PBS hat keine eigene `pve-firewall` – Netzwerk-Einschränkung läuft über nftables/iptables oder Switch-ACLs

❌ Backup-Traffic und produktive Live-Migration dauerhaft über dieselbe geschäftskritische Bandbreite ohne QoS/Trennung laufen lassen
❌ PBS-Datastore-Zugriff ungeschützt aus dem gesamten Management-Netz erlauben

---

## Häufige Fehler & Lösungen

**PVE-Node erreicht PBS nicht auf Port 8007, obwohl beide im selben VLAN hängen sollten**
```
Ursache: PVE-Node hat kein Interface im Backup-VLAN angelegt, Traffic versucht trotzdem dorthin zu routen.
Lösung:  Subinterface (vmbrX.VLAN-ID) auf dem PVE-Node anlegen oder PBS im Management-Netz ansprechen.
```

**`unable to open datastore` trotz korrekter IP**
```
Ursache: Firewall/ACL blockiert Port 8007 zwischen PVE-Node und PBS.
Lösung:  nft/iptables-Regeln prüfen, openssl s_client-Test (Schritt 7) wiederholen.
```

---

## Weiterführende Links

| Ressource | URL |
|---|---|
| PBS Admin Guide | https://pbs.proxmox.com/docs/pbs-admin-guide.html |
| PBS Network Configuration | https://pbs.proxmox.com/docs/installation.html |
