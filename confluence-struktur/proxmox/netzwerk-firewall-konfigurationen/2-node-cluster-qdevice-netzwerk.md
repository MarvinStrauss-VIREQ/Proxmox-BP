<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: Proxmox - Netzwerk- & Firewall-Konfigurationen -->
<!-- Title: Proxmox - 2 Node Cluster + QDevice - Netzwerk -->
<!-- Label: pve-netzwerk, pve-cluster, best-practice, checkliste -->

# 2 Node Cluster + QDevice – Netzwerk

> **PVE 9.x (Debian 13 Trixie)** · Letzte Aktualisierung: 2026-06
> Basis: [Kapitel 02 – Netzwerk & Clustering](../../../kapitel/02-netzwerk-clustering.md), Abschnitt 4
> Verwandte Seiten: [Backup Server - Netzwerk](backup-server-netzwerk.md) · [Firewall](firewall.md) · [Port-Referenz](port-referenz.md)

---

## Wann anwenden?

- Zwei vollwertige PVE-Nodes sollen Cluster-Funktionalität bekommen (Live-Migration, HA), ein dritter Hypervisor-Node ist wirtschaftlich nicht sinnvoll
- VIREQ-Standard: Das **QDevice läuft auf dem PBS-Server**, der ohnehin im Projekt eingeplant ist – kein dritter Server nötig

> **Nicht anwenden** für ≥3 Nodes mit Ceph → [Ceph Cluster - Netzwerk](ceph-cluster-netzwerk.md). Ceph braucht eigenen Quorum-Mechanismus und mindestens 3 Nodes.

---

## Überblick

**Das Quorum-Problem bei 2 Nodes:** Fällt einer der beiden Nodes aus, hat der verbleibende Node nur 1 von 2 Stimmen (50 %) – kein Quorum, VMs werden nicht automatisch neu gestartet. Ein QDevice löst das, indem es eine halbe zusätzliche Stimme einbringt:

```
pve01 (1 Stimme) ↔ pve02 (1 Stimme)
        ↑                 ↑
        └────────────────┘
                ↓
         PBS / QDevice (0,5)

pve01 fällt aus:  pve02 (1) + PBS (0,5) = 1,5 ≥ 1  → Quorum ✅
pve02 fällt aus:  pve01 (1) + PBS (0,5) = 1,5 ≥ 1  → Quorum ✅
PBS fällt aus:    pve01 (1) + pve02 (1) = 2 ≥ 2     → Quorum ✅
```

> **Hyper-V-Vergleich:** QDevice ≈ File Share Witness im Failover-Cluster-Manager – mit dem entscheidenden Unterschied, dass der PBS-Server gleichzeitig auch Backups verwaltet. Kein dritter Server, keine zusätzliche Lizenz.

---

## Voraussetzungen

- Beide PVE-Nodes: [PVE Erstinstallation](../installationen/pve-erstinstallation.md) und [Post Installation Check](../installationen/post-installation-check.md) abgeschlossen
- PBS-Server vorhanden und erreichbar (→ [Backup Server Erstinstallation](../installationen/backup-server-erstinstallation.md)). Falls noch nicht installiert: **zuerst** PBS aufsetzen, da der QDevice-Dienst dort mitläuft
- Hostnamen/FQDNs auf beiden Nodes korrekt in `/etc/hosts` (cluster-kritisch)
- Gleiche PVE-Version auf beiden Nodes

**NIC-Planung (3–4 NICs empfohlen):**

| NIC-Rolle | Funktion |
|---|---|
| 1× Management | Web-UI, SSH, API, Corosync Ring 1 (Fallback) |
| 1× Corosync (dediziert) | Corosync Ring 0 – primärer Cluster-Heartbeat |
| 1–2× VM-Traffic | Bond (active-backup) + VLAN-aware Bridge für Kunden-VMs |

> Corosync braucht **kein** Bonding für Redundanz – es kann selbst zwischen mehreren konfigurierten Netzen (Ring 0/Ring 1) umschalten, wenn eines ausfällt. Das ist der Grund für "dediziert" statt "gebündelt" bei der Corosync-NIC.

---

## Schritt für Schritt

### 1. IP-Schema festlegen

```
192.168.1.0/24    → Management + Corosync Ring 1 (vmbr0)
10.10.10.0/29     → Corosync Ring 0 (dediziert, kein Routing)
192.168.2.0/24     → VM-Traffic (ggf. mehrere VLANs)
```

### 2. Netzwerkkonfiguration auf pve01

`/etc/network/interfaces`:

```
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual

auto eno2
iface eno2 inet manual

auto eno3
iface eno3 inet manual

# Management-Bridge + Corosync Ring 1
auto vmbr0
iface vmbr0 inet static
    address 192.168.1.10/24
    gateway 192.168.1.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0

# Corosync Ring 0 – dedizierte NIC, kein Routing nötig
auto vmbr1
iface vmbr1 inet static
    address 10.10.10.1/29
    bridge-ports eno2
    bridge-stp off
    bridge-fd 0

# VM-Traffic, VLAN-aware
auto vmbr2
iface vmbr2 inet manual
    bridge-ports eno3
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094

dns-nameservers 192.168.1.1
```

**pve02:** identische Struktur, `vmbr0: 192.168.1.11/24`, `vmbr1: 10.10.10.2/29`.

```bash
ifreload -a
ping -c5 10.10.10.2   # von pve01 aus, Corosync-Konnektivität prüfen
```

> Bei nur 2 verfügbaren NICs: Corosync Ring 0 auf `vmbr1` weglassen und stattdessen nur Ring 1 über `vmbr0` nutzen – funktional, aber ohne Redundanz auf Corosync-Ebene. Bei Kundenprojekten mit Wachstumsperspektive lieber gleich auf 3+ NICs planen.

### 3. Cluster erstellen

Immer auf dem **ersten** Node:

```bash
# Auf pve01:
pvecm create FIRMACLUSTER \
    --link0 address=10.10.10.1 \
    --link1 address=192.168.1.10

pvecm status
```

### 4. Zweiten Node beitreten lassen

Der Join-Befehl läuft auf dem **neuen** Node, nicht auf dem bestehenden Cluster:

```bash
# Auf pve02:
pvecm add 192.168.1.10 \
    --link0 address=10.10.10.2 \
    --link1 address=192.168.1.11

pvecm status
pvecm nodes
```

### 5. QDevice auf dem PBS-Server einrichten

```bash
# Auf dem PBS-Server:
apt update && apt install -y corosync-qnetd
systemctl enable --now corosync-qnetd
systemctl status corosync-qnetd
```

### 6. QDevice im Cluster registrieren

```bash
# Auf pve01:
pvecm qdevice setup 192.168.1.50
# SSH-Fingerprint des PBS bestätigen, Root-Passwort des PBS eingeben

# Erwartung: 3 Votes (pve01 + pve02 + QDevice)
pvecm status | grep -A5 "Votequorum"
```

### 7. Firewall-Freigaben für QDevice und Corosync

Das QDevice kommuniziert über TCP 5403 mit beiden PVE-Nodes (vollständige Regeln → [Firewall](firewall.md), Ports → [Port-Referenz](port-referenz.md)):

```ini
# Auszug cluster.fw – 2-Node + QDevice
[IPSET cluster-mgmt]
192.168.1.10  # pve01
192.168.1.11  # pve02

[IPSET pbs-qdevice]
192.168.1.50  # PBS / QDevice

[RULES]
IN ACCEPT -source +dc/cluster-mgmt   -p udp -dport 5405 -log nolog # Corosync Ring 0
IN ACCEPT -source +dc/cluster-mgmt   -p udp -dport 5406 -log nolog # Corosync Ring 1
IN ACCEPT -source +dc/pbs-qdevice    -p tcp -dport 5403 -log nolog # QDevice (Net)
IN ACCEPT -source +dc/cluster-mgmt   -p tcp -dport 60000:60050 -log nolog # Live-Migration
```

### 8. Quorum- und Failover-Test

```bash
# Status auf beiden Nodes:
pvecm status
corosync-quorumtool -s

# Failover-Test: einen Node herunterfahren
ssh root@pve02 "shutdown -h now"

# Auf pve01 prüfen: Quorum sollte mit pve01(1) + QDevice(0,5) = 1,5 weiterhin gegeben sein
pvecm status
```

### 9. Dokumentation

IP-Schema, QDevice-Standort (welcher PBS), Corosync-Ring-Belegung und Failover-Testergebnis dokumentieren.

---

## Best Practices

✅ Dedizierte NIC für Corosync Ring 0 – nie die Management-NIC alleine tragen lassen
✅ Zwei Corosync-Ringe konfigurieren (link0 + link1) statt auf Bonding für Corosync zu setzen
✅ PBS immer als QDevice einplanen statt einen dritten reinen Quorum-Server zu beschaffen
✅ Direkte NIC-zu-NIC-Verbindung für Corosync Ring 0, wenn nur 2 Nodes ohne Switch dazwischen nötig sind

⚠️ Cluster-Name nach dem Erstellen nicht mehr änderbar – vorher mit dem Kunden klären
⚠️ `pvecm add` braucht SSH-Root-Zugriff auf den bestehenden Node

❌ QDevice auf einem der beiden PVE-Nodes selbst betreiben (fällt mit dem Node aus, bringt nichts)
❌ Corosync über eine geroutete WAN-Verbindung führen

---

## Häufige Fehler & Lösungen

**`pvecm add` schlägt fehl: authentication key mismatch**
```
Ursache: Unterschiedliche Corosync-Keys.
Lösung:  scp root@pve01:/etc/corosync/authkey /etc/corosync/authkey
```

**Cluster zeigt "no quorum" nach Neustart eines Nodes**
```
Ursache: QDevice nicht erreichbar.
Prüfen:  pvecm status, systemctl status corosync-qdevice, ping PBS-IP
Lösung:  systemctl restart corosync-qdevice (PVE-Node) / corosync-qnetd (PBS)
```

---

## Weiterführende Links

| Ressource | URL |
|---|---|
| Cluster Manager | https://pve.proxmox.com/wiki/Cluster_Manager |
| QDevice Setup | https://pve.proxmox.com/wiki/Cluster_Manager#_corosync_external_vote_support |
