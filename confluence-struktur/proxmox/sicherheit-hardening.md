<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: Proxmox -->
<!-- Title: Proxmox - Sicherheit & Hardening -->
<!-- Label: pve-sicherheit, best-practice, checkliste -->

# Sicherheit & Hardening

> **PVE 9.x (Debian 13 Trixie)** · Letzte Aktualisierung: 2026-06
> Quelle: adaptiert aus Kapitel 05 – Sicherheit & Hardening.
> Verwandte Seiten: [Firewall](netzwerk-firewall-konfigurationen/firewall.md) · [Post installation check](installationen/post-installation-check.md)

---

## Wann anwenden?

- Server wird in Produktion gesetzt (vor erstem Kundeneinsatz)
- SSH-Zugang absichern, 2FA einrichten, eigene TLS-Zertifikate statt selbstsigniertem Zertifikat
- Rollen und Berechtigungen für Techniker-Accounts anlegen

---

## Überblick

Nach der Basisinstallation bestehen folgende Sicherheitslücken: SSH-Root-Login mit Passwort aktiv, selbstsigniertes TLS-Zertifikat, keine 2FA auf dem Root-Account, alle Ports offen (keine Host-Firewall). Diese Seite schließt diese Lücken systematisch.

---

## 1. SSH absichern

### SSH-Key-Authentifizierung

```bash
# Auf dem Admin-PC:
ssh-keygen -t ed25519 -C "admin@firma.de" -f ~/.ssh/id_pve
ssh-copy-id -i ~/.ssh/id_pve.pub root@192.168.1.10
ssh -i ~/.ssh/id_pve root@192.168.1.10
```

### SSH-Konfiguration härten

```bash
cat >> /etc/ssh/sshd_config << 'EOF'

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

sshd -t && echo "Konfiguration gültig"
systemctl restart ssh
```

> ❌ **Kritisch:** SSH-Verbindung mit Key in einer **neuen** Terminal-Session testen, bevor die bestehende geschlossen wird. Sonst droht Aussperrung.

---

## 2. Firewall-Basis

Vollständige Anleitung inklusive IPsets und Security Groups → [Firewall](netzwerk-firewall-konfigurationen/firewall.md). Kurzform:

```bash
pvesh set /cluster/firewall/options --enable 1
```

```
# Node → Firewall → Add (nur Admin-Netz erlauben)
IN ACCEPT src=192.168.10.0/24 dport=8006 proto=TCP  # Web-UI
IN ACCEPT src=192.168.10.0/24 dport=22   proto=TCP  # SSH
Node → Firewall → Options → Input Policy: DROP
```

> 📸 **Screenshot machen:** Node → Firewall → Options mit Input Policy = DROP – zeigt den kritischsten Schalter der ganzen Seite, bei dem ein Fehler zum Aussperren führen kann.

> ✅ Die PVE-Firewall lässt `established`/`related` automatisch durch.

---

## 3. 2-Faktor-Authentifizierung (TOTP)

```
Web-UI: root-Account → Two Factor → Add
Type: TOTP, Issuer: Proxmox VIREQ
QR-Code mit Authenticator-App scannen (Bitwarden, Authy, Google Authenticator)
```

> 📸 **Screenshot machen:** Datacenter → Permissions → Two Factor (oder direkt am User) → Add-Dialog mit dem QR-Code-Schritt (Werte/QR vor dem Speichern unkenntlich machen oder mit einem Test-Account aufnehmen!) – hilft neuen Technikern, den Ablauf nachzuvollziehen, ohne ihn live am Kundensystem zu testen.

```bash
pveum user tfa add root@pam --type totp
```

> ⚠️ **Recovery-Code offline sichern!** Ohne Token und Recovery-Code kein Login mehr möglich.

---

## 4. TLS-Zertifikate

### Let's Encrypt (ACME, öffentlich erreichbar)

```
Web-UI: Node → Certificates → ACME → Add Account
E-Mail: admin@firma.de, Directory: Let's Encrypt (production)
Domains → Add: pve01.firma.de
Plugin: Standalone (Port 80 offen) oder DNS-Plugin
→ Order Certificates
```

> 📸 **Screenshot machen:** Node → Certificates → ACME-Tab mit angelegtem ACME-Account und der Domain-Liste vor dem "Order"-Klick – zeigt, wo Standalone vs. DNS-Plugin gewählt wird (häufigste Verwechslung bei Erstkonfiguration).

### Eigenes Zertifikat (interne CA)

```bash
cp firma-cert.pem /etc/pve/local/pveproxy-ssl.pem
cp firma-key.pem  /etc/pve/local/pveproxy-ssl.key
systemctl restart pveproxy
```

---

## 5. Rollen und Berechtigungen

### Techniker-Account anlegen

```bash
pveum user add techniker1@pve \
    --password PASSWORT \
    --firstname Max --lastname Techniker \
    --email m.techniker@firma.de

pveum aclmod / --user techniker1@pve --role PVEAdmin
```

> 📸 **Screenshot machen:** Datacenter → Permissions → Users → Add-Dialog sowie direkt danach Datacenter → Permissions → Add (ACL) mit Rolle PVEAdmin auf Pfad `/` – zwei Screenshots, weil das in der Praxis die zwei am häufigsten verwechselten Dialoge sind (User anlegen ≠ Berechtigung vergeben).

| Rolle | Beschreibung |
|---|---|
| `Administrator` | Alles (= root) |
| `PVEAdmin` | VM + Node, kein User-Management |
| `PVEVMAdmin` | VMs verwalten, Snapshots |
| `PVEVMUser` | VMs starten/stoppen, Console |
| `PVEAuditor` | Nur lesen |
| `PVEDatastoreAdmin` | Storage-Verwaltung |

### API-Token (für Automatisierung / PDM / Monitoring)

```bash
pveum user token add monitoring@pve monitoring-token \
    --expire 0 \
    --privsep 1   # 1 = eigene ACLs, 0 = gleiche Rechte wie User

pveum aclmod / --token monitoring@pve!monitoring-token --role PVEAuditor
# Token-Secret wird einmalig angezeigt – sofort sichern!
```

> 📸 **Screenshot machen:** Den einmaligen Token-Secret-Dialog direkt nach dem Anlegen (mit einem Wegwerf-Test-Token, NICHT mit einem echten Secret!) – das ist der wichtigste Moment im ganzen Workflow, weil das Secret danach nie wieder angezeigt wird.

---

## Best Practices

✅ SSH-Keys auf allen Admin-Workstations, dann `PasswordAuthentication` deaktivieren
✅ Eigener Account pro Techniker – kein geteilter Root
✅ API-Token mit minimalen Rechten für Automatisierung
✅ 2FA auf Root-Account unmittelbar nach Grundkonfiguration

⚠️ TOTP-Recovery offline sichern
⚠️ Firewall erst aktivieren wenn Regeln vollständig – sonst Aussperrung
⚠️ SSH-Härtung erst nach erfolgreichem Key-Test

❌ Root-Account für tägliche Arbeit verwenden
❌ API-Token mit Admin-Rechten für Drittsysteme

---

## Häufige Fehler & Lösungen

**SSH nach Härtung gesperrt**
```
Ursache: PasswordAuthentication deaktiviert ohne Key-Test.
Lösung:  Über iDRAC/iLO einloggen, PasswordAuthentication temporär wieder aktivieren.
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
|---|---|
| PVE Firewall | https://pve.proxmox.com/wiki/Firewall |
| User Management | https://pve.proxmox.com/pve-docs/pve-admin-guide.html#chapter_user_management |
| ACME / Let's Encrypt | https://pve.proxmox.com/wiki/Certificate_Management |
