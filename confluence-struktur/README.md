# Confluence-Struktur – Mapping

Dieser Ordner spiegelt die Seitenstruktur des Confluence-Space `KNOWLEDGEBASE` (Seitenbaum unter "Proxmox") 1:1, damit der spätere Sync per `mark` direkt aus diesen Dateien laufen kann. Jede Datei trägt das übliche `mark`-Front-Matter (Space/Parent/Title/Label) und ist als eigenständige Confluence-Seite gedacht – nicht als Verweis auf die Kapitel-Dateien unter `kapitel/`.

**Wichtig:** Seiten, die in Confluence laut Marvin bereits Inhalt haben (`Hardware Specs`, `Post installation check`, `PVE Erstinstallation`, `Firewall`, `Port-Referenz`), wurden hier aus dem bestehenden Repo-Inhalt (v.a. Kapitel 01 und 07) neu zusammengestellt. **Vor dem Sync gegen den aktuellen Confluence-Stand prüfen**, um nichts zu überschreiben, was dort schon anders/aktueller gepflegt ist.

## Seitenbaum ↔ Datei ↔ Quelle

| Confluence-Seite | Datei | Quelle |
|---|---|---|
| Proxmox - Hardware Specs | `proxmox/hardware-specs.md` | Platzhalter – kein Repo-Quellinhalt, siehe Hinweis in der Datei |
| Proxmox - Installationen | *(Strukturseite, keine eigene Datei)* | – |
| Proxmox - Backup Server Erstinstallation | `proxmox/installationen/backup-server-erstinstallation.md` | Kapitel 03, Abschnitt 1–3 |
| Proxmox - Datacenter Manager Erstinstallation | `proxmox/installationen/datacenter-manager-erstinstallation.md` | Kapitel 04, Abschnitt 1–2 |
| Proxmox - Post installation check | `proxmox/installationen/post-installation-check.md` | Kapitel 01, Checkliste + Abschnitt 4 |
| Proxmox - PVE Erstinstallation (Stand PVE 9.1) | `proxmox/installationen/pve-erstinstallation.md` | Kapitel 01, Abschnitt 1–3 |
| Proxmox - Netzwerk- & Firewall-Konfigurationen | *(Strukturseite, keine eigene Datei)* | – |
| Proxmox - 2 Node Cluster + QDevice - Netzwerk | `proxmox/netzwerk-firewall-konfigurationen/2-node-cluster-qdevice-netzwerk.md` | NEU, Basis: Kapitel 02 Abschnitt 4 |
| Proxmox - backup Server - Netzwerk | `proxmox/netzwerk-firewall-konfigurationen/backup-server-netzwerk.md` | NEU |
| Proxmox - Ceph Cluster - Netzwerk | `proxmox/netzwerk-firewall-konfigurationen/ceph-cluster-netzwerk.md` | NEU, Basis: Kapitel 07 NIC-Layout + Kapitel 09 |
| Proxmox - Firewall | `proxmox/netzwerk-firewall-konfigurationen/firewall.md` | Kapitel 07 (IPsets, Security Groups, Aktivierung) |
| Proxmox - Port-Referenz | `proxmox/netzwerk-firewall-konfigurationen/port-referenz.md` | Kapitel 07 (Port-Tabelle, CLI-Referenz, simulate) |
| Proxmox - Single Node - Netzwerk | `proxmox/netzwerk-firewall-konfigurationen/single-node-netzwerk.md` | NEU |
| Proxmox - Storage & Backup | `proxmox/storage-backup.md` | Kapitel 03 (Datastore, Jobs, Retention, GC, Restore) |
| Proxmox - LVM & ZFS | `proxmox/lvm-zfs.md` | Kapitel 01 Abschnitt 2.4/5 + ergänzt |

## Status

- ✅ Vollständig neu & im Detail ausformuliert: die vier Netzwerk-Anwendungsfälle (Single Node, 2-Node+QDevice, Backup Server, Ceph Cluster)
- 🔄 Aus bestehenden Kapiteln adaptiert, noch nicht final mit Confluence abgeglichen: Hardware Specs, Installationen-Unterseiten, Firewall, Port-Referenz, Storage & Backup, LVM & ZFS
