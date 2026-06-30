<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: Proxmox - Installationen -->
<!-- Title: Proxmox - Datacenter Manager Erstinstallation -->
<!-- Label: pve-installation, checkliste -->

# Datacenter Manager Erstinstallation

> **Proxmox Datacenter Manager (PDM) 1.x** · Letzte Aktualisierung: 2026-06
> Quelle: adaptiert aus Kapitel 04, Abschnitt 1–2.
> Voraussetzung: mindestens ein funktionierender Proxmox-Cluster

---

## Wann anwenden?

- Mehrere Proxmox-Cluster sollen zentral verwaltet werden
- Live-Migration von VMs zwischen verschiedenen Clustern benötigt
- Einheitliche Oberfläche für mehrere Standorte oder Kunden

> **Nicht erforderlich** für Einzelcluster.

---

## Überblick

Der PDM ist ein zentrales Management-Tool für mehrere PVE-Cluster: zentrale Web-Oberfläche, Cross-Cluster Live-Migration, zentrale Ressourcenübersicht, gemeinsame Benutzerverwaltung (LDAP/AD).

> **VMware-Vergleich:** PDM ≈ vSphere vCenter (vereinfacht) – ein Management-Server für mehrere Hypervisor-Cluster. Kein Backup-Tool (→ PBS), kein Monitoring, kein Ersatz für die PVE Web-UI bei Node-spezifischen Konfigurationen.

---

## Voraussetzungen

- Alle Cluster betriebsbereit
- PDM-Server: eigener Linux-Server oder VM (Debian 13 Trixie)
- Netzwerkzugang zur PVE-API (Port 8006) aller Nodes
- Für Cross-Cluster-Migration: Shared Storage oder kompatible Storage-Typen

---

## 1. Installation

```bash
apt update && apt install -y curl gnupg2

curl -fsSL https://enterprise.proxmox.com/debian/proxmox-release-trixie.gpg \
    | gpg --dearmor > /etc/apt/trusted.gpg.d/proxmox-release-trixie.gpg

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

## 2. Cluster in PDM registrieren

```
PDM Web-UI: Remote Clusters → Add Remote
Name:        Cluster-Berlin
Host:        192.168.1.10 (IP eines Cluster-Nodes)
Port:        8006
Username:    root@pam
```

> ✅ Empfohlen: dedizierten API-Token statt Root-Passwort verwenden.

---

## Best Practices

✅ PDM auf dedizierter VM (nicht auf einem verwalteten PVE-Node) betreiben
✅ API-Tokens statt Root-Passwort für Cluster-Anbindung
✅ PDM-Zugriff auf Management-VLAN beschränken

⚠️ PDM-Ausfall hat keinen Einfluss auf die Cluster – PVE läuft eigenständig weiter
⚠️ Cross-Cluster-Migration überträgt Speicher übers Netzwerk – Bandbreite vorher prüfen

---

## Weiterführende Links

| Ressource | URL |
|---|---|
| PDM Admin Guide | https://pdm.proxmox.com/pdm-admin-guide.html |
