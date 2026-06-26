# CLAUDE.md — Proxmox Best Practices Repository

## Projektkontext

Marvin Strauß, IT-Consultant bei VIREQ, baut diese Wissensbasis auf, um Proxmox als Alternative zu Hyper-V bei Kundenprojekten zu positionieren. Das Team nutzt aktuell Hyper-V mit Hornetsecurity VM Backup (ehem. Altaro) und ein 2-Server-Cluster-Modell.

## Repository-Struktur

```
Proxmox-BP/
├── README.md
├── CLAUDE.md
├── index.json
├── configurator/
│   └── configurator.html
└── kapitel/
    ├── 01-grundkonfiguration.md      # PVE Installation & Post-Install
    ├── 02-netzwerk-clustering.md     # Netzwerk, Corosync, QDevice
    ├── 03-backup-strategien.md       # PBS, vzdump, Retention
    ├── 04-datacenter-manager.md      # PDM, Multi-Site
    ├── 05-sicherheit-hardening.md    # SSH, Firewall, 2FA, Zertifikate
    ├── 06-monitoring-alerting.md     # NinjaOne, Alerts, Monitoring
    ├── 07-firewall-ipsets.md         # Firewall, IPsets, Security Groups
    ├── 08-sdn.md                     # SDN, VNets, VLAN Zone
    ├── 09-ceph-storage.md            # Ceph Setup, Pools, RBD
    └── 10-high-availability.md       # HA Manager, Fencing, Failover
```

## Zielversionen

- Proxmox VE: 9.x (Debian 13 Trixie) — Hinweise für 8.x (Bookworm) jeweils markiert
- Proxmox Backup Server: 4.x
- Proxmox Datacenter Manager: 1.x
- Ceph: 19.x (Squid)

## Kapitel-Template

Jedes Kapitel folgt diesem Aufbau:

```markdown
# Kapitel N: Titel

> Version, letzte Aktualisierung, verwandte Kapitel

## Wann anwenden?

## Überblick
Kurzbeschreibung + Hyper-V-Vergleich wo sinnvoll

## Voraussetzungen

## Abschnitte mit Schritt-für-Schritt-Anleitungen

## Best Practices
✅ Empfohlen:
⚠️ Achtung:
❌ Vermeiden:

## Häufige Fehler & Lösungen

## Weiterführende Links
```

## Entwicklungsrichtlinien

1. **Sprache:** Immer Deutsch
2. **„Wann anwenden?"-Sektion:** Pflicht in jedem Kapitel
3. **Hyper-V-Vergleiche:** Einfügen wo sinnvoll
4. **Befehle erklären:** Was der Befehl tut, nicht nur wie er aussieht
5. **Echte Werte:** Keine Platzhalter wie `<your-ip>` ohne Erklärung
6. **Versionierung:** PVE 9.x als Primärversion, PVE 8.x als Hinweis
7. **Suchindex:** `index.json` bei jedem neuen Kapitel aktualisieren

## Schlüsselerkenntnisse (nicht vergessen)

- PBS dient gleichzeitig als **Backup-Server UND QDevice** für 2-Node-Cluster
- **Hardware-RAID-Controller** inkompatibel mit ZFS → HBA IT-Mode notwendig
- **Intel-NICs** bevorzugen (igb/ixgbe), Broadcom vorab validieren, Realtek nur Management
- **Enterprise-SSDs mit PLP** sind Pflicht auf PBS-Systemen
- **Hosts-Datei** muss auf Management-IP zeigen (nicht 127.x.x.x) → Cluster-kritisch
- **RAM-Formel PVE:** 2 GB OS + 1 GB/TB ZFS ARC + VM-RAM
- **RAM-Formel PBS:** 4 GB OS + 1 GB/TB Dedup-Index + 1 GB/TB ZFS ARC
- **NIC-Anzahl:** Single: 2×, 2-Node: 3–4×, 3+Node/Ceph: 4–6×
- **PDM** = Proxmox Datacenter Manager (nicht Proxmox Mail Gateway!)
- **apt full-upgrade** statt apt upgrade auf PVE
- Hornetsecurity unterstützt Proxmox erst ab PVE 9.0+, kein LXC-Support

## Firewall-Spezifika (pve-firewall)

### Syntax-Regeln – verifiziert gegen PVE 9.x

| Parameter | Falsch ❌ | Richtig ✅ |
|-----------|----------|----------|
| Kommentare | `-comment "text"` | `# text` am Zeilenende |
| Protokoll | `-proto tcp` | `-p tcp` |
| Datacenter-IPsets | `+ipset-name` | `+dc/ipset-name` |

Quelle: UI-generierte Regeln haben Format `IN ACCEPT -source +dc/ipset -p tcp -dport 22 -log nolog # Kommentar`

### pve-firewall CLI Referenz

```bash
pve-firewall compile          # Regeln kompilieren + Syntaxprüfung (kein Output = OK)
pve-firewall status           # Firewall-Status anzeigen
pve-firewall restart          # Service neu starten
pve-firewall localnet         # Lokale Netzwerk-Info anzeigen

# Regel-Simulation (OHNE Aktivierung testen)
pve-firewall simulate \
  --from outside --to host \
  --source <quell-ip> \
  --dport <port> \
  --protocol tcp|udp

# pve-firewall stop  ← ACHTUNG: entfernt ALLE Regeln, Node ungeschützt!
```

### simulate – Beispiele für diesen Cluster

```bash
# SSH von Admin erlaubt?
pve-firewall simulate --from outside --to host --source 172.30.7.50 --dport 22 --protocol tcp

# Web UI von Admin erlaubt?
pve-firewall simulate --from outside --to host --source 172.30.7.50 --dport 8006 --protocol tcp

# Corosync Ring 0 zwischen Nodes?
pve-firewall simulate --from outside --to host --source 10.10.20.12 --dport 5405 --protocol udp

# Ceph Monitor von Storage-Netz?
pve-firewall simulate --from outside --to host --source 10.20.20.12 --dport 3300 --protocol tcp
```

### Security Groups

- Security Groups: nur für **VMs/CT** — nicht für Host-Ebene
- Definition in `cluster.fw` als `[group sg-name]`-Abschnitt
- Referenz in VM-Firewall: `GROUP sg-name`
- IPsets in Security Groups brauchen `+dc/` Prefix (VM-Scope → Datacenter-Scope)

## Referenz-Cluster (3-Node, Ceph)

Dieser Cluster dient als Blueprint-Grundlage:

| Node | Management (vmbr0) | Corosync Ring 0 (ens19) | Ceph (bond1) |
|------|--------------------|------------------------|--------------|
| pve1 | 172.30.7.31/16 | 10.10.20.11/24 | 10.20.20.11/24 |
| pve2 | 172.30.7.32/16 | 10.10.20.12/24 | 10.20.20.12/24 |
| pve3 | 172.30.7.33/16 | 10.10.20.13/24 | 10.20.20.13/24 |

Netzwerk-Layout pro Node: nic0→vmbr0 (Web/Ring1), ens19 (Ring0), bond0(ens20+21)→vmbr1 (VMs, VLAN-aware), bond1(ens22+23) (Ceph, balance-rr)

Ceph: 12 OSDs (4/Node), 3 MON, 2 MGR, Ceph 19.2.3 (Squid), Pool vm-store (size=3, min_size=2)

## Konfigurator-Logik

| Eingabe | Empfohlene Kapitel |
|---------|-------------------|
| Neue Installation | 01, 02, 03 |
| Cluster (2-Node) | 02 (QDevice) + 03 (PBS als QDevice) |
| Cluster (3+ Nodes) | 02, 09 (Ceph) |
| Backup einrichten | 03 |
| Multi-Site / PDM | 04 |
| Security-Audit | 05, 07 |
| Monitoring fehlt | 06 |
| VM-Netzwerke / VLANs | 08 (SDN) |
| Firewall einrichten | 07 |
| HA / Failover | 09 + 10 |

## Offene Punkte

- [ ] PDF-Export-Funktion für Offline-Nutzung
- [ ] Separate Dokumentation für PVE 8.x vs. 9.x
- [ ] Migrations-Playbook: Hyper-V → Proxmox
- [ ] Hardware-Checkliste für Pre-Sales
- [ ] TCO-Vergleich Proxmox vs. Hyper-V vs. VMware
- [ ] Kapitel 11: VM Best Practices (CPU-Typ, Balloon, NUMA)
