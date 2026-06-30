# Confluence-Struktur – Mapping

Dieser Ordner spiegelt die Seitenstruktur des Confluence-Space `KNOWLEDGEBASE` (Seitenbaum unter "Proxmox") 1:1, damit der spätere Sync per `mark` direkt aus diesen Dateien laufen kann. Jede Datei trägt das übliche `mark`-Front-Matter (Space/Parent/Title/Label) und ist als eigenständige Confluence-Seite gedacht – nicht als Verweis auf die Kapitel-Dateien unter `kapitel/`.

**Wichtig:** Seiten, die in Confluence laut Marvin bereits Inhalt haben (`Hardware Specs`, `Post installation check`, `PVE Erstinstallation`, `Firewall`, `Port-Referenz`), wurden hier aus dem bestehenden Repo-Inhalt neu zusammengestellt. **Vor dem Sync gegen den aktuellen Confluence-Stand prüfen**, um nichts zu überschreiben, was dort schon anders/aktueller gepflegt ist.

Marvins ursprüngliche Confluence-Seitenstruktur deckte nur Installation, Netzwerk/Firewall, Storage/Backup und LVM/ZFS ab. Nach Rücksprache wurden zusätzlich sechs Lücken identifiziert und ergänzt: High Availability, Sicherheit & Hardening, Monitoring & Alerting und SDN waren im Repo bereits dokumentiert (Kapitel 05/06/08/10), hatten aber keine Confluence-Seite. Migration von Hyper-V, Updates & Patch-Management, Resource Pools & Multi-Tenancy, VM-Templates & Cloud-Init und PCI/GPU-Passthrough existierten bisher gar nicht und wurden neu recherchiert und geschrieben.

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
| Proxmox - 2 Node Cluster + QDevice - Netzwerk | `proxmox/netzwerk-firewall-konfigurationen/2-node-cluster-qdevice-netzwerk.md` | Basis: Kapitel 02 Abschnitt 4 |
| Proxmox - backup Server - Netzwerk | `proxmox/netzwerk-firewall-konfigurationen/backup-server-netzwerk.md` | NEU |
| Proxmox - Ceph Cluster - Netzwerk | `proxmox/netzwerk-firewall-konfigurationen/ceph-cluster-netzwerk.md` | Basis: Kapitel 07 NIC-Layout + Kapitel 09 |
| Proxmox - Firewall | `proxmox/netzwerk-firewall-konfigurationen/firewall.md` | Kapitel 07 (IPsets, Security Groups, Aktivierung) |
| Proxmox - Port-Referenz | `proxmox/netzwerk-firewall-konfigurationen/port-referenz.md` | Kapitel 07 (Port-Tabelle, CLI-Referenz, simulate) |
| Proxmox - Single Node - Netzwerk | `proxmox/netzwerk-firewall-konfigurationen/single-node-netzwerk.md` | NEU |
| Proxmox - SDN | `proxmox/netzwerk-firewall-konfigurationen/sdn.md` | Kapitel 08 |
| Proxmox - Storage & Backup | `proxmox/storage-backup.md` | Kapitel 03 (Datastore, Jobs, Retention, GC, Restore) |
| Proxmox - LVM & ZFS | `proxmox/lvm-zfs.md` | Kapitel 01 Abschnitt 2.4/5 + ergänzt |
| Proxmox - High Availability | `proxmox/high-availability.md` | Kapitel 10 |
| Proxmox - Sicherheit & Hardening | `proxmox/sicherheit-hardening.md` | Kapitel 05 |
| Proxmox - Monitoring & Alerting | `proxmox/monitoring-alerting.md` | Kapitel 06 |
| Proxmox - Ceph Storage | `proxmox/ceph-storage.md` | Kapitel 09 (Setup-Ebene: MON/MGR/OSD/Pool/CRUSH) |
| Proxmox - Migration von Hyper-V | `proxmox/migration-von-hyperv.md` | NEU, recherchiert (offizielle PVE-Doku + Praxisquellen) |
| Proxmox - Updates & Patch-Management | `proxmox/updates-patch-management.md` | NEU, recherchiert |
| Proxmox - Resource Pools & Multi-Tenancy | `proxmox/resource-pools-multi-tenancy.md` | NEU, recherchiert |
| Proxmox - VM-Templates & Cloud-Init | `proxmox/vm-templates-cloud-init.md` | NEU, recherchiert |
| Proxmox - PCI/GPU-Passthrough | `proxmox/pci-gpu-passthrough.md` | NEU, recherchiert (ergänzt Kapitel 01 IOMMU-Grundlagen) |

## Status

- ✅ Vollständig neu & im Detail ausformuliert: die vier Netzwerk-Anwendungsfälle (Single Node, 2-Node+QDevice, Backup Server, Ceph Cluster) sowie die fünf komplett neuen Themenseiten (Migration von Hyper-V, Updates & Patch-Management, Resource Pools & Multi-Tenancy, VM-Templates & Cloud-Init, PCI/GPU-Passthrough)
- 🔄 Aus bestehenden Kapiteln adaptiert, noch nicht final mit Confluence abgeglichen: Hardware Specs, Installationen-Unterseiten, Firewall, Port-Referenz, Storage & Backup, LVM & ZFS, High Availability, Sicherheit & Hardening, Monitoring & Alerting, SDN, Ceph Storage

## Bekannte offene Punkte

- **Sicherheit & Hardening** und **Monitoring & Alerting** sind aktuell direkte Kinder von "Proxmox" – falls dafür eine eigene Struktursektion analog zu "Installationen" oder "Netzwerk- & Firewall-Konfigurationen" gewünscht ist, muss das Parent-Feld in den jeweiligen Dateien angepasst werden.
- **Ceph Storage** und **Ceph Cluster - Netzwerk** überschneiden sich bewusst nicht: Erstere deckt MON/MGR/OSD/Pool-Setup ab, letztere NICs/Bonds/IP-Schema. Bei Bedarf zu einer Seite zusammenführen.
- Index.json und CLAUDE.md im Repo-Root spiegeln diese neue Struktur noch nicht.
