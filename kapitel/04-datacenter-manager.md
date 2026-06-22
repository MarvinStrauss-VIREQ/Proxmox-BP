# Kapitel 4: Datacenter Manager (PDM)

> **Version:** Proxmox Datacenter Manager 1.x · **Letzte Aktualisierung:** 2026-06  
> **Voraussetzung:** Mindestens ein funktionierender Proxmox-Cluster (→ Kapitel 2)

---

## Wann anwenden?

> - Mehrere Proxmox-Cluster zentral verwalten
> - Live-Migration von VMs **zwischen** verschiedenen Clustern
> - Einheitliche Oberfläche für mehrere Standorte oder Kunden
>
> **Nicht erforderlich** für Einzelcluster.

---

## Überblick

Der **Proxmox Datacenter Manager (PDM)** ist ein zentrales Management-Tool für mehrere PVE-Cluster. Stabil seit PDM 1.0 (Dezember 2025).

**Kernfunktionen:**
- Zentrale Web-Oberfläche für alle Cluster
- **Cross-Cluster Live-Migration** (VM zwischen Clustern ohne Downtime)
- Zentrale Ressourcenübersicht (CPU, RAM, Storage über alle Cluster)
- Gemeinsame Benutzerverwaltung (LDAP/AD-Integration)

> **VMware-Vergleich:** PDM ≈ vSphere vCenter (vereinfacht) – ein Management-Server für mehrere Hypervisor-Cluster.

**Was PDM nicht ist:** Kein Backup-Tool (→ PBS), kein Monitoring (→ Kapitel 6), kein Ersatz für PVE Web-UI bei Node-spezifischen Konfigurationen.

---

## Voraussetzungen

- Alle Cluster betriebsbereit (→ Kapitel 1 + 2)
- PDM-Server: eigener Linux-Server oder VM (Debian 13 Trixie)
- PDM benötigt Netzwerkzugang zur PVE-API (Port 8006) aller Nodes
- Für Cross-Cluster-Migration: Shared Storage oder kompatible Storage-Typen

---

## 1. PDM Installation

```bash
# Auf dem PDM-Server (Debian 13 Trixie):
apt update && apt install -y curl gnupg2

# Proxmox-Key hinzufügen:
curl -fsSL https://enterprise.proxmox.com/debian/proxmox-release-trixie.gpg \
    | gpg --dearmor > /etc/apt/trusted.gpg.d/proxmox-release-trixie.gpg

# PDM No-Subscription-Repository:
cat >> /etc/apt/sources.list << 'EOF'
deb http://download.proxmox.com/debian/pdm trixie pdm-no-subscription
EOF

apt update && apt install -y proxmox-datacenter-manager
systemctl enable --now proxmox-datacenter-manager
```

```
Web-UI: https://PDM-IP:8443
Benutzer: root, Realm: Linux PAM
```

---

## 2. Cluster in PDM registrieren

```
PDM Web-UI: Remote Clusters → Add Remote
Name:        Cluster-Berlin
Host:        192.168.1.10 (IP eines Cluster-Nodes)
Port:        8006
Fingerprint: (automatisch abrufen oder manuell)
Username:    root@pam
Password:    [Root-Passwort]
```

> ✅ Empfohlen: Dedizierten API-Token statt Root-Passwort verwenden (→ Kapitel 5, Abschnitt API-Token).

---

## 3. Cross-Cluster Live-Migration

> PDM ermöglicht Migration laufender VMs zwischen verschiedenen Clustern ohne Downtime.

**Voraussetzungen:**
- PDM hat Zugriff auf beide Cluster
- Ausreichend RAM/CPU auf Ziel-Node
- Kompatible Storage-Typen oder Shared Storage
- Netzwerkverbindung zwischen Quell- und Ziel-Cluster

```
PDM Web-UI:
Remote Clusters → [Quell-Cluster] → VMs/CTs
VM auswählen → Migrate → Remote Migration
Ziel-Cluster:     [Ziel-Cluster]
Ziel-Node:        pve01
Ziel-Storage:     vmpool
Online-Migration: ✓ (Live-Migration ohne Downtime)
```

---

## 4. Zentrale Benutzerverwaltung

```
PDM Web-UI: Access Control → Users → Add
Username: techniker1
Realm:    proxmox-datacenter oder LDAP/AD
Role:     PVEVMUser (nur VMs) / PVEAdmin (vollständig)
Scope:    Alle Cluster oder spezifischer Cluster/Node
```

---

## Best Practices

✅ PDM auf dedizierter VM (nicht auf verwalteten PVE-Nodes) betreiben  
✅ API-Tokens statt Root-Passwort für Cluster-Anbindung  
✅ PDM-Zugriff auf Management-VLAN beschränken

⚠️ PDM-Ausfall hat keinen Einfluss auf Cluster (PVE läuft eigenständig weiter)  
⚠️ Cross-Cluster-Migration überträgt Speicher übers Netzwerk – Bandbreite prüfen

---

## Weiterführende Links

| Ressource | URL |
|-----------|-----|
| PDM Admin Guide | https://pdm.proxmox.com/pdm-admin-guide.html |
| PDM Release Notes | https://www.proxmox.com/en/about/company-details/press-releases |

---

**→ Nächstes Kapitel:** [Kapitel 5 – Sicherheit & Hardening](05-sicherheit-hardening.md)  
**← Vorheriges Kapitel:** [Kapitel 3 – Backup-Strategien](03-backup-strategien.md)
