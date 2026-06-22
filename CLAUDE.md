# CLAUDE.md — Proxmox Best Practices Repository

## Projektkontext

Marvin Strauß, IT-Consultant bei VIREQ, baut diese Wissensbasis auf, um Proxmox als Alternative zu Hyper-V bei Kundenprojekten zu positionieren. Das Team nutzt aktuell Hyper-V mit Hornetsecurity VM Backup (ehem. Altaro) und ein 2-Server-Cluster-Modell.

## Repository-Struktur

```
Proxmox-BP/
├── README.md                          # Übersicht, Navigation, Schlüsselkonzepte
├── CLAUDE.md                          # Projektanweisungen für Claude
├── index.json                         # Suchindex für alle Kapitel
├── configurator/
│   └── configurator.html             # Interaktiver Konfigurator (offline-fähig)
└── kapitel/
    ├── 01-grundkonfiguration.md      # PVE Installation & Post-Install
    ├── 02-netzwerk-clustering.md     # Netzwerk, Corosync, QDevice
    ├── 03-backup-strategien.md       # PBS, vzdump, Retention
    ├── 04-datacenter-manager.md      # PDM, Multi-Site
    ├── 05-sicherheit-hardening.md    # SSH, Firewall, 2FA, Zertifikate
    └── 06-monitoring-alerting.md     # Grafana, InfluxDB, Alerts
```

## Zielversionen

- Proxmox VE: 9.x (Debian 13 Trixie) — Hinweise für 8.x (Bookworm) jeweils markiert
- Proxmox Backup Server: 4.x
- Proxmox Datacenter Manager: 1.x

## Kapitel-Template

Jedes Kapitel folgt diesem Aufbau:

```markdown
# Kapitel N: Titel

> Version, letzte Aktualisierung, verwandte Kapitel

## Wann anwenden?
> Kurzentscheidung: wann ist dieses Kapitel relevant?

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
2. **„Wann anwenden?“-Sektion:** Pflicht in jedem Kapitel
3. **Hyper-V-Vergleiche:** Einfügen wo sinnvoll — hilft bei Kunden-Adoption-Gesprächen
4. **Befehle erklären:** Was der Befehl tut, nicht nur wie er aussieht
5. **Echte Werte:** Keine Platzhalter wie `<your-ip>` ohne Erklärung
6. **Versionierung:** PVE 9.x als Primärversion, PVE 8.x als Hinweis
7. **Offline-fähig:** Kein CDN im Konfigurator, keine externen Abhängigkeiten
8. **Suchindex:** `index.json` bei jedem neuen Kapitel aktualisieren

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

## Konfigurator-Logik

| Eingabe | Empfohlene Kapitel |
|---------|-------------------|
| Neue Installation | 01, 02, 03 |
| Cluster (2-Node) | 02 (QDevice) + 03 (PBS als QDevice) |
| Cluster (3+ Nodes) | 02 |
| Backup einrichten | 03 |
| Multi-Site / PDM | 04 |
| Security-Audit | 05 |
| Monitoring fehlt | 06 |

## Offene Punkte

- [ ] PDF-Export-Funktion für Offline-Nutzung
- [ ] Separate Dokumentation für PVE 8.x vs. 9.x
- [ ] Kapitel 7: Ceph-Storage (geplant)
- [ ] Migrations-Playbook: Hyper-V → Proxmox (geplant)
- [ ] Hardware-Checkliste für Pre-Sales (geplant)
- [ ] TCO-Vergleich Proxmox vs. Hyper-V vs. VMware (geplant)
