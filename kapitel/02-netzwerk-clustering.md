# Kapitel 2: Netzwerk & Clustering

> **Version:** Proxmox VE 9.x · **Letzte Aktualisierung:** 2026-06  
> **Voraussetzung:** [Kapitel 1 – PVE Grundkonfiguration](01-grundkonfiguration.md) auf jedem Node abgeschlossen

---

## Wann anwenden?

> - Mehrere Proxmox-Nodes sollen zu einem Cluster zusammengeschlossen werden
> - 2-Node-Cluster ohne dritten physischen Server (PBS als QDevice)
> - Dedizierte Netzwerke für Cluster- und VM-Traffic
> - Bond/LACP oder VLAN-Trunks konfigurieren

---

## Überblick

Proxmox nutzt **Corosync** als Cluster-Kommunikationsprotokoll (UDP-Multicast/Unicast). Für Quorum bei Node-Ausfall wird entweder ein dritter Node oder ein **QDevice** benötigt.

| Topologie | Nodes | Quorum | Empfehlung |
|-----------|-------|--------|----------|
| Single Node | 1 | Kein Quorum | Testlab |
| 2-Node + PBS QDevice | 2 + PBS | ✅ via QDevice | KMU-Standard |
| 3-Node | 3 | ✅ Nativ | Ab 3 Nodes |
| 3-Node + Ceph | 3 | ✅ Nativ + Ceph | Hyper-Converged |

> **Hyper-V-Vergleich:** Corosync ≈ WSFC. QDevice ≈ File Share Witness. Der Vorteil: PBS übernimmt **gleichzeitig** Backup und QDevice – kein dritter physischer Server nötig.

---

## Voraussetzungen

- Kapitel 1 auf allen Nodes abgeschlossen (Hosts-Datei, NTP, Updates)
- Hostnames und FQDNs definiert (`pve01.firma.local`, `pve02.firma.local`)
- Für 2-Node: PBS-Server vorhanden (→ Kapitel 3)

**NIC-Planung:**

| Cluster-Typ | NICs | Belegung |
|------------|------|----------|
| Single Node | 2 | 1× Management+VM, 1× Reserve |
| 2-Node | 3–4 | 1× Management, 1× Corosync, 1–2× VM |
| 3+-Node/Ceph | 4–6 | 1× Mgmt, 1–2× Corosync, 1× Ceph, 1–2× VM |

---

## 1. Netzwerk-Architektur

### 1.1 Netzwerk-Trennung (empfohlen)

```
192.168.1.0/24   → Management (Web-UI, SSH, API)
10.10.10.0/29    → Corosync  (Cluster-Traffic, kein Routing)
192.168.2.0/24   → VM-Traffic (ggf. mehrere VLANs)
```

**Warum trennen?** Cluster-Traffic beeinträchtigt keine VM-Performance, bessere Sicherheitsisolierung.

### 1.2 Netzwerkkonfiguration 2-Node-Cluster

**pve01 `/etc/network/interfaces`:**
```
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual

auto eno2
iface eno2 inet manual

# Management-Bridge
auto vmbr0
iface vmbr0 inet static
    address 192.168.1.10/24
    gateway 192.168.1.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094

# Corosync-Bridge (nur Cluster-Traffic)
auto vmbr1
iface vmbr1 inet static
    address 10.10.10.1/29
    bridge-ports eno2
    bridge-stp off
    bridge-fd 0

dns-nameservers 192.168.1.1
```

**pve02:** Gleiche Struktur, `vmbr0: 192.168.1.11/24`, `vmbr1: 10.10.10.2/29`

```bash
ifreload -a
```

### 1.3 Bond/LACP (optional, für Hochverfügbarkeit)

```
auto eno1
iface eno1 inet manual
    bond-master bond0

auto eno2
iface eno2 inet manual
    bond-master bond0

auto bond0
iface bond0 inet manual
    bond-slaves eno1 eno2
    bond-miimon 100
    bond-mode 802.3ad
    bond-xmit-hash-policy layer2+3

auto vmbr0
iface vmbr0 inet static
    address 192.168.1.10/24
    gateway 192.168.1.1
    bridge-ports bond0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094
```

> ⚠️ LACP erfordert Switch-seitige Konfiguration (Port-Channel). Ohne Switch-Unterstützung: `bond-mode active-backup`.

---

## 2. Cluster erstellen

> Cluster **immer auf dem ersten Node** erstellen.

```bash
# Auf pve01:
pvecm create FIRMACLUSTER \
    --link0 address=10.10.10.1 \
    --link1 address=192.168.1.10

# link0 = primärer Corosync-Ring (Cluster-NIC)
# link1 = sekundärer Ring / Fallback (Management-NIC)

# Status prüfen:
pvecm status
pvecm nodes
```

**Gesunder Cluster-Status:**
```
Quorum information
------------------
Quorate:          Yes
Expected votes:   1 (vor QDevice-Setup)
```

---

## 3. Zweiten Node hinzufügen

> ⚠️ **Join-Befehl auf dem NEUEN Node** ausführen, nicht auf dem Cluster.

```bash
# Auf pve02:
pvecm add 192.168.1.10 \
    --link0 address=10.10.10.2 \
    --link1 address=192.168.1.11

# Root-Passwort von pve01 wird abgefragt

# Status prüfen (auf beiden Nodes):
pvecm status
pvecm nodes
```

> Nach dem Join: VMs und Storage automatisch im Web-UI beider Nodes sichtbar.

---

## 4. QDevice für 2-Node-Cluster (PBS als QDevice)

> **Problem:** Bei 2 Nodes hat ein verbleibender Node nach Ausfall des anderen nur 1 von 2 Stimmen (50%) – kein Quorum, VMs werden nicht neu gestartet.
>
> **Lösung:** PBS erhält 0,5 Stimmen. Bei Ausfall eines Nodes: verbliebener Node (1) + PBS (0,5) = 1,5 ≥ 1 ✅
>
> **Hyper-V-Vergleich:** PBS als QDevice ≈ File Share Witness auf NAS – mit dem entscheidenden Vorteil, dass PBS gleichzeitig Backups verwaltet.

```
pve01 (1 Stimme) ↔ pve02 (1 Stimme)
        ↑                 ↑
        └───────────────┘
                ↓
         PBS/QDevice (0,5)

pve01 aus: pve02(1) + PBS(0.5) = 1.5 ≥ 1 ✅
pve02 aus: pve01(1) + PBS(0.5) = 1.5 ≥ 1 ✅
PBS aus:  pve01(1) + pve02(1) = 2 ≥ 2 ✅
```

### 4.1 QDevice auf PBS einrichten

```bash
# Auf dem PBS-Server:
apt update && apt install -y corosync-qnetd
systemctl enable --now corosync-qnetd
systemctl status corosync-qnetd
```

### 4.2 QDevice im Cluster konfigurieren

```bash
# Auf pve01:
pvecm qdevice setup 192.168.1.50

# SSH-Fingerprint des PBS bestätigen
# Root-Passwort des PBS eingeben
# corosync-qdevice wird automatisch installiert

# Ergebnis prüfen (3 Votes erwartet):
pvecm status | grep -A5 "Votequorum"
```

---

## 5. Corosync optimieren und prüfen

```bash
# Konfiguration anzeigen:
cat /etc/pve/corosync.conf

# Token-Timeout anpassen (Standard: 5000ms):
pvecm updateconfig --totem_token 6000

# Latenz testen (Ziel: <1ms bei direkter Verbindung):
ping -c 100 10.10.10.2 | tail -1

# Corosync-Log auf Fehler prüfen:
journalctl -u corosync --since "1 hour ago" | grep -iE "error|fail"

# Quorum-Status:
corosync-quorumtool -s
```

---

## Best Practices

✅ Dediziertes NIC für Corosync — nie Management-NIC teilen  
✅ Zwei Corosync-Ringe konfigurieren (link0 + link1 als Fallback)  
✅ PBS immer als QDevice für 2-Node — kein dritter Server nötig  
✅ Direkte NIC-zu-NIC-Verbindung für Corosync bei 2-Node (ohne Switch)  
✅ Alle Nodes vor Cluster-Join: NTP, Hosts-Datei, gleiche PVE-Version

⚠️ Cluster-Name nach Erstellen nicht mehr änderbar — gut überlegen  
⚠️ `pvecm add` braucht SSH-Root-Zugriff auf bestehenden Node  
⚠️ Nach Cluster-Join: Storage-Konfiguration des neuen Nodes wird überschrieben

❌ QDevice auf einem der PVE-Nodes betreiben (fällt mit dem Node aus)  
❌ Corosync über geroutete WAN-Verbindung führen  
❌ Unterschiedliche PVE-Versionen im Cluster

---

## Häufige Fehler & Lösungen

**`pvecm add` schlägt fehl: authentication key mismatch**
```
Ursache: Verschiedene Corosync-Keys.
Lösung:  scp root@pve01:/etc/corosync/authkey /etc/corosync/authkey
```

**Cluster zeigt "no quorum" nach Neustart**
```
Ursache: QDevice nicht erreichbar.
Prüfen:  pvecm status, systemctl status corosync-qdevice, ping PBS-IP
Lösung:  systemctl restart corosync-qdevice  (PVE-Nodes)
         systemctl restart corosync-qnetd     (PBS)
```

**Zweiter Node nicht im Web-UI sichtbar**
```
Ursache: FQDN-Auflösung schlägt fehl.
Prüfen:  hostname --fqdn, ping pve02 (von pve01)
Lösung:  /etc/hosts auf beiden Nodes korrigieren (→ Kapitel 1, 3.4)
```

---

## Weiterführende Links

| Ressource | URL |
|-----------|-----|
| Cluster Manager | https://pve.proxmox.com/wiki/Cluster_Manager |
| QDevice Setup | https://pve.proxmox.com/wiki/Cluster_Manager#_corosync_external_vote_support |
| HA Manager | https://pve.proxmox.com/wiki/High_Availability |

---

**→ Nächstes Kapitel:** [Kapitel 3 – Backup-Strategien](03-backup-strategien.md)  
**← Vorheriges Kapitel:** [Kapitel 1 – PVE Grundkonfiguration](01-grundkonfiguration.md)
