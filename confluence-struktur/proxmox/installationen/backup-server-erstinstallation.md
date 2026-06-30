<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: Proxmox - Installationen -->
<!-- Title: Proxmox - Backup Server Erstinstallation -->
<!-- Label: pve-installation, checkliste -->

# Backup Server Erstinstallation

> **PBS 4.x (Debian 13 Trixie)** · Letzte Aktualisierung: 2026-06
> Quelle: adaptiert aus Kapitel 03, Abschnitt 1–3.
> Verwandte Seiten: [backup Server - Netzwerk](../netzwerk-firewall-konfigurationen/backup-server-netzwerk.md) · [Storage & Backup](../storage-backup.md) · [2 Node Cluster + QDevice - Netzwerk](../netzwerk-firewall-konfigurationen/2-node-cluster-qdevice-netzwerk.md)

---

## Wann anwenden?

- Bei jedem neuen Kundenprojekt – PBS wird laut VIREQ-Standard immer mitgeplant
- Wenn PBS zusätzlich als QDevice für einen 2-Node-Cluster dienen soll

---

## Überblick

| Komponente | Minimum | Produktion |
|---|---|---|
| CPU | 2 Kerne | 4+ Kerne |
| RAM | 4 GB | 8–16 GB |
| System-Disk | 32 GB SSD | 64 GB SSD |
| Backup-Storage | Beliebig | **Enterprise-SSD mit PLP** |
| NIC | 1 × 1G | 1 × 10G |

> ❌ Consumer-SSDs ohne PLP (Power Loss Protection) können bei Stromausfall den ZFS-Pool korrumpieren – auf PBS sind Enterprise-SSDs Pflicht.

**RAM-Formel:** `RAM = 4 GB (OS) + 1 GB × [Dedup-Index in TB] + 1 GB × [ZFS-Pool in TB]` – Beispiel 10 TB Pool: 4 + 10 + 10 = **24 GB RAM**.

---

## 1. Installation

```bash
# ISO: https://www.proxmox.com/de/downloads/proxmox-backup-server/iso
# Installation identisch zu PVE (→ PVE Erstinstallation, Abschnitt 2)
# Empfehlung: ZFS Mirror für System-Disk
# Web-UI nach Install: https://PBS-IP:8007

cat > /etc/apt/sources.list.d/pbs-enterprise.list << 'EOF'
# deb https://enterprise.proxmox.com/debian/pbs trixie pbs-enterprise
EOF

cat >> /etc/apt/sources.list << 'EOF'
deb http://download.proxmox.com/debian/pbs trixie pbs-no-subscription
EOF

apt update && apt full-upgrade -y
```

> PBS 3.x (Bookworm): `trixie` → `bookworm`

## 2. Netzwerk konfigurieren

Identische Bonding-/Bridge-Logik wie bei PVE-Nodes – vollständige Anleitung inklusive IP-Schema und optionalem Backup-VLAN → [backup Server - Netzwerk](../netzwerk-firewall-konfigurationen/backup-server-netzwerk.md).

## 3. Backup-Datastore anlegen

Datastore-Erstellung, Backup-Jobs, Retention und Wiederherstellung → [Storage & Backup](../storage-backup.md).

## 4. Optional: PBS als QDevice für 2-Node-Cluster

Falls dieser PBS-Server zusätzlich Quorum-Funktion übernehmen soll → [2 Node Cluster + QDevice - Netzwerk](../netzwerk-firewall-konfigurationen/2-node-cluster-qdevice-netzwerk.md), Schritt 5–6.

---

## Best Practices

✅ PBS auf dedizierter Hardware, nicht als VM auf dem gesicherten Cluster
✅ Enterprise-SSDs mit PLP für ZFS-Backup-Pool
✅ RAM-Formel vor Beschaffung berechnen, nicht erst nach Auslieferung

❌ PBS als VM auf dem gesicherten Cluster betreiben
❌ Backup- und VM-Storage auf denselben physischen Disks

---

## Weiterführende Links

| Ressource | URL |
|---|---|
| PBS Admin Guide | https://pbs.proxmox.com/docs/pbs-admin-guide.html |
