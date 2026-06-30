<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: Proxmox -->
<!-- Title: Proxmox - Resource Pools & Multi-Tenancy -->
<!-- Label: pve-cluster, best-practice, referenz -->

# Resource Pools & Multi-Tenancy

> **PVE 9.x (Debian 13 Trixie)** · Letzte Aktualisierung: 2026-06
> Verwandte Seiten: [Sicherheit & Hardening](sicherheit-hardening.md) · [SDN](netzwerk-firewall-konfigurationen/sdn.md)

---

## Wann anwenden?

- Mehrere Kunden oder Teams teilen sich einen Proxmox-Cluster und sollen nur ihre eigenen Ressourcen sehen/verwalten
- Techniker-Accounts sollen nur auf bestimmte VMs/CTs/Storage zugreifen dürfen, nicht auf den gesamten Cluster
- Reporting/Abrechnung pro Kunde soll über eine logische Gruppierung vereinfacht werden

> **Proxmox ist kein "fertiges" Multi-Tenant-Produkt** wie OpenStack – Isolation entsteht durch die Kombination aus Resource Pools (RBAC), SDN-Zonen (Netzwerk-Trennung) und ggf. getrennten Storage-Bereichen pro Mandant. Das muss bewusst zusammengesetzt werden.

---

## Überblick

Ein Resource Pool ist eine benannte Sammlung von VMs, Containern und Storage-Volumes. Statt jedes Objekt einzeln zu berechtigen, wird eine Berechtigung einmal auf den Pool gesetzt – alle enthaltenen Ressourcen erben sie.

```
Ohne Pool:  Berechtigung pro VM einzeln vergeben (100, 101, 102, ... )
Mit Pool:   1× Pool "kunde-a" anlegen → VMs zuordnen → 1× ACL auf den Pool
```

### Grenzen, die man kennen muss

| Was Pools/RBAC **können** | Was sie **nicht** können (Stand PVE 9.x) |
|---|---|
| Sichtbarkeit/Verwaltung auf definierte VMs/Storage beschränken | Harte CPU/RAM-Quotas pro Pool durchsetzen – ein Mandant kann theoretisch beliebig viele/große VMs anlegen |
| Netzwerk-Isolation über SDN-Zonen/VNets ergänzen | Storage-Quotas nur, wenn das darunterliegende Storage selbst Quotas unterstützt (z. B. eigenes ZFS-Dataset/Ceph-Pool pro Mandant) |

> Für echte CPU/RAM-Hard-Limits pro Mandant gibt es in PVE aktuell keinen nativen Mechanismus – das ist ein bekannter, von Proxmox selbst bestätigter Gap. Bei harten Kapazitätsanforderungen eher getrennte Cluster (über Datacenter Manager) oder externe Guardrails (Hookscripts) einplanen.

---

## Bausteine der Berechtigungsarchitektur

| Baustein | Funktion |
|---|---|
| **User** | Einzelaccount, einer oder mehreren Gruppen zuordenbar |
| **Group** | Bündelt User für gemeinsame Rechtevergabe |
| **Role** | Satz von Privilegien (vordefiniert: `PVEAdmin`, `PVEVMUser`, `PVEAuditor` …, oder custom) |
| **Pool** | Gruppiert VMs/CTs/Storage zu einer logischen Einheit |
| **ACL** | Verknüpft User/Group + Role + Pfad (z. B. `/pool/kunde-a`) |

Permissions sind pfadbasiert und werden vererbt (`propagate`-Flag); spezifischere Pfade überschreiben allgemeinere. `NoAccess` gewinnt immer gegen andere Rollen auf demselben Pfad.

---

## Schritt für Schritt: Mandanten-Setup mit Pools

### 1. Gruppen und Benutzer anlegen

Auch über **Web UI: Datacenter → Permissions → Groups/Users → Add** möglich.

> 📸 **Screenshot machen:** Datacenter → Permissions → Groups → Add-Dialog sowie Users → Add-Dialog – zwei Bilder, weil das die Grundbausteine sind, auf die alles Weitere aufbaut.

```bash
pveum group add kunde-a-team --comment "Techniker Kunde A"
pveum user add techniker-a@pve --password PASSWORT --comment "Techniker Kunde A"
pveum user modify techniker-a@pve --groups kunde-a-team
```

### 2. Resource Pool anlegen und Ressourcen zuordnen

Auch über **Web UI: Datacenter → Permissions → Pools → Create** möglich.

> 📸 **Screenshot machen:** Datacenter → Permissions → Pools → Create-Dialog, sowie direkt danach den geöffneten Pool mit der Mitglieder-Liste (VMs + Storage) – zeigt, wie Ressourcen einem Pool nachträglich hinzugefügt werden.

```bash
pveum pool add kunde-a --comment "Alle VMs und Storage von Kunde A"
pveum pool modify kunde-a --vms 100,101,102
pveum pool modify kunde-a --storage kunde-a-storage
```

### 3. Berechtigung auf den Pool setzen

> 📸 **Screenshot machen:** Datacenter → Permissions → Add-Dialog mit Pfad `/pool/kunde-a`, Rolle `PVEVMAdmin`, Gruppe `kunde-a-team` – das ist der Dialog, der am häufigsten falsch ausgefüllt wird (Pfad vs. Pool-Auswahl verwechseln).

```bash
pveum acl modify /pool/kunde-a --roles PVEVMAdmin --groups kunde-a-team
```

Neue VMs, die später dem Pool hinzugefügt werden, erben automatisch dieselbe Berechtigung – kein manuelles Nachpflegen pro VM nötig.

### 4. Storage-Trennung pro Mandant (für echte Kapazitätskontrolle)

```bash
# Eigenes ZFS-Dataset mit Quota pro Mandant
zfs create -o quota=2T vmpool/kunde-a
pvesm add zfspool kunde-a-storage --pool vmpool/kunde-a --content images,rootdir
```

> Dasselbe Prinzip funktioniert mit Ceph: ein eigener Pool pro Mandant statt eines geteilten `vm-store`-Pools, wenn Storage-Quotas durchgesetzt werden müssen.

### 5. Netzwerk-Isolation ergänzen

Pools regeln nur Sichtbarkeit/Verwaltung – für echte Netzwerktrennung zwischen Mandanten zusätzlich eine eigene SDN-VNet/Zone pro Kunde anlegen (→ [SDN](netzwerk-firewall-konfigurationen/sdn.md)) und per Firewall-Regeln absichern.

### 6. Node-Sichtbarkeit einschränken (optional)

```bash
pveum acl modify /nodes/pve3 --roles NoAccess --groups kunde-a-team --propagate 1
```

> Funktioniert zuverlässig für Sichtbarkeit der Node-Verwaltung selbst. Es gibt **keinen** nativen Mechanismus, um einen Mandanten technisch auf bestimmte Nodes für VM-Platzierung zu "zwingen" – das muss organisatorisch (Naming-Konvention, Storage-Verfügbarkeit nur auf den vorgesehenen Nodes) gelöst werden.

---

## API-Token statt Passwort für Automatisierung

```bash
pveum user token add techniker-a@pve automation --privsep 1
pveum acl modify /pool/kunde-a --roles PVEVMUser --tokens techniker-a@pve!automation
```

`--privsep 1` sorgt dafür, dass der Token nur die explizit zugewiesenen Rechte hat – nicht automatisch die vollen Rechte des zugehörigen Users.

---

## Best Practices

✅ Nie den `root@pam`-Account teilen – ein Account pro Techniker, Gruppen für Rechtevergabe
✅ Pools nach Kunde/Projekt benennen, nicht nach technischen Details (ändern sich seltener)
✅ Eigenes ZFS-Dataset oder Ceph-Pool pro Mandant, wenn Storage-Quotas Pflicht sind
✅ SDN-Zonen + Firewall-Regeln immer mitdenken – Pools alleine isolieren kein Netzwerk
✅ API-Token mit `privsep=1` für jede Automatisierung/Integration

⚠️ CPU/RAM sind aktuell "Best Effort" pro Pool – keine harten Limits ohne externe Guardrails (Hookscripts) oder separate Cluster
⚠️ `NoAccess` auf einem übergeordneten Pfad überschreibt spezifischere Rollen weiter unten – bei unerwartet fehlenden Berechtigungen zuerst dort prüfen

❌ Mehrere Mandanten ohne Netzwerk-Trennung im selben VLAN/Bridge betreiben
❌ Sich bei harten Kapazitätszusagen gegenüber Kunden allein auf Pools verlassen

---

## Häufige Fehler & Lösungen

**User sieht trotz Pool-Mitgliedschaft keine VMs**
```
Ursache: NoAccess-Regel auf einem übergeordneten Pfad (z.B. /vms) überschreibt die Pool-ACL.
Lösung:  pveum acl list prüfen; explizite NoAccess-Einträge auf höheren Pfaden entfernen oder Reihenfolge der ACLs verstehen (Pools werden zuerst zu /vms/XXX und /storage/YYY expandiert).
```

**Mandant belegt deutlich mehr Storage als vereinbart**
```
Ursache: Kein Quota auf dem zugrundeliegenden Storage gesetzt, nur Pool-Sichtbarkeit konfiguriert.
Lösung:  Eigenes ZFS-Dataset/Ceph-Pool mit Quota pro Mandant statt geteiltem Storage.
```

---

## Weiterführende Links

| Ressource | URL |
|---|---|
| PVE User Management (RBAC, Pools) | https://pve.proxmox.com/pve-docs/pve-admin-guide.html#chapter_user_management |
