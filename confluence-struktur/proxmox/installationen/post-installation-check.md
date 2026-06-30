<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: Proxmox - Installationen -->
<!-- Title: Proxmox - Post installation check -->
<!-- Label: pve-installation, checkliste -->

# Post installation check

> **PVE 9.x (Debian 13 Trixie)** · Letzte Aktualisierung: 2026-06
> Quelle: adaptiert aus Kapitel 01 (Checkliste + Abschnitt 4). **Vor Sync gegen bestehenden Confluence-Stand prüfen.**
> Verwandte Seiten: [PVE Erstinstallation](pve-erstinstallation.md)

---

## Wann anwenden?

- Direkt nach jeder PVE-Erstinstallation, vor jedem Cluster-Join und vor Produktivbetrieb

---

## Checkliste

> ⚠️ = cluster-kritisch · 🔗 = nur bei Cluster · *(optional)* = situationsabhängig

| # | Aufgabe |
|---|---|
| 1 | Lizenz eintragen **oder** No-Subscription-Repo aktivieren |
| 2 | System-Update + Neustart: `apt full-upgrade -y && reboot` |
| 3 | System-Check: `pveversion -v`, `ip addr`, `ping 8.8.8.8` |
| 4 | Hostname + `/etc/hosts` prüfen ⚠️ |
| 5 | Netzwerk prüfen: Bond, vmbr0 |
| 6 | NTP / Zeitzone prüfen |
| 7 | E-Mail-Benachrichtigungen einrichten |
| 8 | 2FA für Root-User einrichten |
| 9 | SSH Key-Auth einrichten |
| 10 | Anmeldedaten in Bitwarden hinterlegen |
| 11 | NinjaOne Agent installieren |
| 12 | ZFS Pool anlegen *(wenn zutreffend)* |
| 13 | ZFS ARC begrenzen *(nur bei ZFS-OS)* |
| 14 | S.M.A.R.T.-Monitoring einrichten |
| 15 | Backup konfigurieren |
| 16 | Firewall Basis aktivieren |
| 17 | IOMMU im Kernel aktivieren *(optional, PCI-Passthrough)* |
| 18 | No-Subscription-Popup entfernen *(optional)* |
| 19 | 🔗 Cluster-Join |
| 20 | 🔗 PBS als QDevice + Backup-Server verbinden |

---

## Schritt für Schritt

### Repository umstellen (No-Subscription)

```bash
cat > /etc/apt/sources.list.d/pve-enterprise.list << 'EOF'
# deb https://enterprise.proxmox.com/debian/pve trixie pve-enterprise
EOF

cat >> /etc/apt/sources.list << 'EOF'

deb http://download.proxmox.com/debian/pve trixie pve-no-subscription
EOF

apt update
```
> PVE 8.x (Bookworm): `trixie` → `bookworm`

### System aktualisieren

```bash
apt full-upgrade -y
reboot
pveversion -v
```

### Hosts-Datei konfigurieren ⚠️ (cluster-kritisch)

> Hostname **muss** auf die Management-IP zeigen – nie auf `127.0.x.x`. Falscher Eintrag = Cluster-Join schlägt fehl.

```bash
cat > /etc/hosts << 'EOF'
127.0.0.1       localhost
192.168.1.10    pve01.kunde.local pve01
EOF

hostname --fqdn   # Muss lauten: pve01.kunde.local
```

### Zeitsynchronisation

```bash
chronyc tracking
timedatectl set-timezone Europe/Berlin
```

### E-Mail-Benachrichtigungen

```
Datacenter → Notifications → Add → SMTP
Host: smtp.office365.com · Port: 587 · Mode: STARTTLS
```

### IOMMU aktivieren (optional, für PCI-Passthrough)

```bash
nano /etc/default/grub
# GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
update-grub
reboot
dmesg | grep -e DMAR -e IOMMU
```

> **Hyper-V-Vergleich:** entspricht Discrete Device Assignment (DDA).

---

## Best Practices

✅ `apt full-upgrade` (nicht `apt upgrade`) – Hosts-Datei vor Cluster-Join prüfen
✅ Passwort sofort in Bitwarden hinterlegen
✅ Reihenfolge der Checkliste einhalten – Firewall erst nach NTP/DNS/Updates aktivieren

❌ `127.0.1.1` in `/etc/hosts`

---

## Häufige Fehler & Lösungen

**`401 Unauthorized` bei `apt update`**
```
Ursache: Enterprise-Repo aktiv, keine Subscription.
Lösung:  No-Subscription-Repo aktivieren (siehe oben).
```

**`hostname --fqdn` gibt `localdomain` zurück**
```
Ursache: /etc/hosts falsch konfiguriert.
Lösung:  Management-IP korrekt eintragen.
```

---

## Weiterführende Links

| Ressource | URL |
|---|---|
| PVE Admin Guide | https://pve.proxmox.com/pve-docs/pve-admin-guide.html |
