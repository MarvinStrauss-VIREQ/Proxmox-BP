# Proxmox Best Practices

> Interne Wissensbasis für Proxmox VE, Proxmox Backup Server (PBS) und Proxmox Datacenter Manager (PDM).  
> Zielgruppe: IT-Techniker und -Berater, die Proxmox-Projekte planen, einrichten und betreiben.

## Kapitelübersicht

| # | Kapitel | Status | Beschreibung |
|---|---------|--------|-------------|
| 1 | [PVE Grundkonfiguration](kapitel/01-grundkonfiguration.md) | ✅ | Installation, Post-Install, ZFS, Storage, Netzwerk |
| 2 | [Netzwerk & Clustering](kapitel/02-netzwerk-clustering.md) | ✅ | Corosync, QDevice, Cluster-Setup, Bond/LACP |
| 3 | [Backup-Strategien](kapitel/03-backup-strategien.md) | ✅ | PBS, vzdump, Retention, QDevice-Doppelrolle |
| 4 | [Datacenter Manager](kapitel/04-datacenter-manager.md) | ✅ | PDM, Multi-Site, Cross-Cluster-Migration |
| 5 | [Sicherheit & Hardening](kapitel/05-sicherheit-hardening.md) | ✅ | SSH, Firewall, 2FA, Zertifikate, Rollen |
| 6 | [Monitoring & Alerting](kapitel/06-monitoring-alerting.md) | ✅ | Notifications, Grafana, InfluxDB, Zabbix |

## Schnellstart

Nicht sicher, welches Kapitel relevant ist? → [Konfigurator öffnen](configurator/configurator.html)

## Schlüsselkonzepte

**Cluster-Topologien:**

| Typ | Nodes | Quorum-Lösung | Empfehlung |
|-----|-------|--------------|----------|
| Single Node | 1 | – | Testlab |
| 2-Node-Cluster | 2 + PBS | PBS als QDevice | ✅ KMU-Standard |
| 3-Node-Cluster | 3 | Nativ | Ab 3 Nodes |
| 3-Node + Ceph | 3 | Nativ + Ceph HA | Hyper-Converged |

**Wichtige Grundregeln:**
- PBS **immer** mitdeployen — Backup-Server + QDevice in einem
- **Hardware-RAID** → muss auf **HBA IT-Mode** konvertiert werden (ZFS-Inkompatibilität)
- **Intel-NICs** bevorzugen — Broadcom vorab validieren, Realtek nur Management
- **Enterprise-SSDs mit PLP** auf PBS-Systemen (Power Loss Protection)

**RAM-Formeln:**
```
PVE-Nodes: 2 GB (OS) + 1 GB × [ZFS-Pool in TB] (ARC) + Σ VM-RAM
PBS:       4 GB (OS) + 1 GB × [Dedup-Index in TB] + 1 GB × [ZFS-Pool in TB]
```

## Hyper-V Übersetzungstabelle

| Proxmox Konzept | Hyper-V Äquivalent | Hinweis |
|----------------|-------------------|--------|
| Corosync Cluster | WSFC (Windows Server Failover Cluster) | Proxmox: konfigurationslos, automatisch |
| QDevice (PBS) | File Share Witness (FSW) | PBS übernimmt gleichzeitig Backup-Funktion |
| Proxmox Backup Server | Backup-VM auf NAS / Hornetsecurity | Nativ integriert, Dedup, Verifizierung |
| vmbr0 (Linux Bridge) | External Virtual Switch | Bridged Networking |
| LXC Container | – (kein direktes Äquivalent) | Leichtgewichtige Linux-Container |
| ZFS Mirror | Windows Storage Spaces Mirror | Self-Healing, Checksummen |
| Proxmox Datacenter Manager | System Center VMM (vereinfacht) | Cross-Cluster-Live-Migration |
| Proxmox VE (KVM) | Hyper-V (Typ-1-Hypervisor) | Direkter Wettbewerber |

## NIC-Anforderungen nach Cluster-Typ

| Cluster-Typ | Min. NICs | Belegung |
|------------|-----------|--------|
| Single Node | 2 | 1× Management/VM, 1× Reserve |
| 2-Node-Cluster | 3–4 | 1× Management, 1× Corosync, 1–2× VM |
| 3+-Node / Ceph | 4–6 | 1× Management, 1–2× Corosync, 1× Ceph, 1–2× VM |

---
*Erstellt und gepflegt mit Unterstützung von Claude (Anthropic). Version: PVE 9.x / PBS 4.x / PDM 1.x*
