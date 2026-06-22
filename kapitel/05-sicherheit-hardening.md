# Kapitel 5: Sicherheit & Hardening

> **Version:** Proxmox VE 9.x · **Letzte Aktualisierung:** 2026-06  
> **Voraussetzung:** [Kapitel 1 – PVE Grundkonfiguration](01-grundkonfiguration.md) abgeschlossen

---

## Wann anwenden?

> - Server wird in Produktion gesetzt (vor erstem Kundeneinsatz)
> - SSH-Zugang absichern
> - 2-Faktor-Authentifizierung einrichten
> - Eigene TLS-Zertifikate statt selbstsigniertem Zertifikat
> - Rollen und Berechtigungen für Techniker-Accounts anlegen

---

## Überblick

Nach der Basisinstallation bestehen folgende Sicherheitslücken:
- SSH-Root-Login mit Passwort aktiv
- Selbstsigniertes TLS-Zertifikat (Browser-Warnung)
- Keine 2FA auf Root-Account
- Alle Ports offen (keine Host-Firewall)

Dieses Kapitel schließt diese Lücken systematisch.

---

## 1. SSH absichern

### 1.1 SSH-Key-Authentifizierung

```bash
# Auf dem Admin-PC:
ssh-keygen -t ed25519 -C "admin@firma.de" -f ~/.ssh/id_pve

# Public Key auf PVE übertragen:
ssh-copy-id -i ~/.ssh/id_pve.pub root@192.168.1.10

# Verbindung mit Key testen:
ssh -i ~/.ssh/id_pve root@192.168.1.10
```

### 1.2 SSH-Konfiguration härten

```bash
cat >> /etc/ssh/sshd_config << 'EOF'

# Sicherheitshärtung
PermitRootLogin prohibit-password
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
AllowTcpForwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
EOF

# Konfiguration validieren:
sshd -t && echo "Konfiguration gültig"

# Neu starten:
systemctl restart ssh
```

> ❌ **Kritisch:** SSH-Verbindung mit Key in einer **neuen** Terminal-Session testen, bevor die bestehende geschlossen wird. Sonst droht Aussperrung.

---

## 2. Proxmox Firewall

### 2.1 Datacenter-Firewall aktivieren

```bash
# Web-UI: Datacenter → Firewall → Options → Firewall: Yes
pvesh set /cluster/firewall/options --enable 1
```

### 2.2 Node-Regeln (nur Admin-Netz erlauben)

```
Web-UI: Node → Firewall → Add

# Management (nur aus Admin-VLAN 192.168.10.0/24):
IN ACCEPT src=192.168.10.0/24 dport=8006 proto=TCP  # Web-UI
IN ACCEPT src=192.168.10.0/24 dport=22   proto=TCP  # SSH
IN ACCEPT src=192.168.10.0/24 dport=3128 proto=TCP  # SPICE

# Corosync (nur zwischen Cluster-Nodes):
IN ACCEPT src=10.10.10.0/29 dport=5405-5406 proto=UDP

# Alles andere blockieren:
Node → Firewall → Options → Input Policy: DROP
```

> ✅ PVE-Firewall lässt `established`/`related` automatisch durch.

---

## 3. 2-Faktor-Authentifizierung (TOTP)

```
Web-UI: root-Account → Two Factor → Add
Type: TOTP
Issuer: Proxmox VIREQ
QR-Code mit Authenticator-App scannen
(Google Authenticator, Bitwarden, Authy)
Code eingeben → Aktivieren
```

```bash
# Via CLI:
pveum user tfa add root@pam --type totp
```

> ⚠️ **Recovery-Code offline sichern!** Ohne Token und Recovery-Code kein Login mehr möglich.

---

## 4. TLS-Zertifikate

### 4.1 Let's Encrypt (ACME, öffentlich erreichbar)

```
Web-UI: Node → Certificates → ACME → Add Account
E-Mail: admin@firma.de
Directory: Let's Encrypt (production)

Domains → Add: pve01.firma.de
Plugin: Standalone (Port 80 offen) oder DNS-Plugin
→ Order Certificates
```

### 4.2 Eigenes Zertifikat (interne CA)

```bash
# Zertifikat und Key hochladen:
# Web-UI: Node → Certificates → Upload Custom Certificate

# Via CLI:
cp firma-cert.pem /etc/pve/local/pveproxy-ssl.pem
cp firma-key.pem  /etc/pve/local/pveproxy-ssl.key
systemctl restart pveproxy
```

---

## 5. Rollen und Berechtigungen

### 5.1 Techniker-Account anlegen

```bash
# Benutzer erstellen:
pveum user add techniker1@pve \
    --password PASSWORT \
    --firstname Max --lastname Techniker \
    --email m.techniker@firma.de

# Vollständige Verwaltung (ohne User-Management):
pveum aclmod / --user techniker1@pve --role PVEAdmin

# Nur VM-Verwaltung:
pveum aclmod / --user techniker1@pve --role PVEVMAdmin
```

**Vordefinierte Rollen:**

| Rolle | Beschreibung |
|-------|-------------|
| `Administrator` | Alles (= root) |
| `PVEAdmin` | VM + Node, kein User-Management |
| `PVEVMAdmin` | VMs verwalten, Snapshots |
| `PVEVMUser` | VMs starten/stoppen, Console |
| `PVEAuditor` | Nur lesen |
| `PVEDatastoreAdmin` | Storage-Verwaltung |

### 5.2 API-Token (für Automatisierung / PDM / Monitoring)

```bash
# Token erstellen:
pveum user token add monitoring@pve monitoring-token \
    --expire 0 \
    --privsep 1   # 1 = eigene ACLs, 0 = gleiche Rechte wie User

# Berechtigungen setzen:
pveum aclmod / --token monitoring@pve!monitoring-token --role PVEAuditor

# Token-Secret wird einmalig angezeigt – sofort sichern!
# Verwendung: Authorization: PVEAPIToken=monitoring@pve!monitoring-token=SECRET
```

---

## Best Practices

✅ SSH-Keys auf allen Admin-Workstations, dann PasswordAuthentication deaktivieren  
✅ Eigener Account pro Techniker — kein geteilter Root  
✅ API-Token mit minimalen Rechten für Automatisierung  
✅ 2FA auf Root-Account unmittelbar nach Grundkonfiguration

⚠️ TOTP-Recovery offline sichern  
⚠️ Firewall erst aktivieren wenn Regeln vollständig – sonst Aussperrung  
⚠️ SSH-Härtung (PasswordAuthentication no) erst nach Key-Test

❌ Root-Account für tägliche Arbeit verwenden  
❌ API-Token mit Admin-Rechten für Drittsysteme

---

## Häufige Fehler & Lösungen

**SSH nach Härtung gesperrt**
```
Ursache: PasswordAuthentication deaktiviert ohne Key-Test.
Lösung:  Über iDRAC/iLO einloggen.
         /etc/ssh/sshd_config: PasswordAuthentication yes (temporär)
         SSH-Keys korrekt einrichten, dann wieder deaktivieren.
```

**TOTP-Code wird abgelehnt**
```
Ursache: Systemzeit weicht ab.
Prüfen:  chronyc tracking | grep "System time"
Lösung:  chronyc makestep
```

---

## Weiterführende Links

| Ressource | URL |
|-----------|-----|
| PVE Firewall | https://pve.proxmox.com/wiki/Firewall |
| User Management | https://pve.proxmox.com/pve-docs/pve-admin-guide.html#chapter_user_management |
| ACME / Let's Encrypt | https://pve.proxmox.com/wiki/Certificate_Management |

---

**→ Nächstes Kapitel:** [Kapitel 6 – Monitoring & Alerting](06-monitoring-alerting.md)  
**← Vorheriges Kapitel:** [Kapitel 4 – Datacenter Manager](04-datacenter-manager.md)
