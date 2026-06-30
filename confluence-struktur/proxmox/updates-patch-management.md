<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: Proxmox -->
<!-- Title: Proxmox - Updates & Patch-Management -->
<!-- Label: pve-cluster, best-practice, checkliste -->

# Updates & Patch-Management

> **PVE 9.x (Debian 13 Trixie)** · Letzte Aktualisierung: 2026-06
> Verwandte Seiten: [High Availability](high-availability.md) · [Ceph Storage](ceph-storage.md)

---

## Wann anwenden?

- Regelmäßiges Patch-Management für produktive PVE-Cluster (monatlich/quartalsweise)
- Major-Version-Upgrades (z. B. PVE 8 → 9)
- Vor und nach jedem Update bei Clustern mit HA und/oder Ceph

> **Nicht anwenden** ohne vorheriges Backup aller VMs/CTs – das gilt uneingeschränkt, auch für Minor-Updates.

---

## Überblick

In einem Proxmox-Cluster werden **niemals alle Nodes gleichzeitig** aktualisiert – das Vorgehen ist ein Rolling Update: ein Node nach dem anderen wird entleert, aktualisiert, neu gestartet und erst dann der nächste angefasst. Das hält Quorum und VM-Verfügbarkeit während des gesamten Vorgangs aufrecht.

```
Falsch:  apt dist-upgrade auf allen 3 Nodes gleichzeitig
         → Quorum-Verlust, VMs auf allen Nodes betroffen

Richtig: pve1 entleeren → Update → Reboot → verifizieren
         → pve2 entleeren → Update → Reboot → verifizieren
         → pve3 entleeren → Update → Reboot → verifizieren
```

> ⚠️ **Niemals `apt upgrade` verwenden** – nur `apt dist-upgrade`. `apt upgrade` berücksichtigt Abhängigkeiten zwischen Paketquellen nicht korrekt und kann das System unbenutzbar machen.

---

## Voraussetzungen

- Vollständiges Backup aller VMs/CTs (→ [Storage & Backup](storage-backup.md))
- Bei Ceph: `ceph status` zeigt `HEALTH_OK` vor Beginn
- Bei HA: Geplante Downtime kommuniziert, falls Failover-Tests parallel laufen sollen
- Alle Nodes auf demselben Ausgangs-Patchlevel – keine gemischten Major-Versionen im Cluster

---

## Routine-Update (Minor, monatlich/quartalsweise)

### Schritt 1: Pro Node – VMs entleeren

```bash
# Alle laufenden VMs eines Nodes live auf andere Nodes migrieren
for vmid in $(pvesh get /nodes/pve2/qemu --output-format json | jq -r '.[].vmid'); do
  qm migrate "$vmid" pve1 --online
done
```

### Schritt 2: Bei HA – Node in Wartungsmodus versetzen

```bash
ha-manager crm-command node-maintenance enable pve2
```

### Schritt 3: Bei Ceph – Flags setzen (verhindert unnötiges Rebalancing während kurzer Downtime)

```bash
ceph status   # muss HEALTH_OK zeigen, bevor irgendetwas angefasst wird
ceph osd set noout
ceph osd set nobackfill
ceph osd set norecover
```

### Schritt 4: Update durchführen

Auch über **Web UI: Node → Updates → Refresh, dann Upgrade** möglich (öffnet eine Shell mit den exakt gleichen Befehlen).

> 📸 **Screenshot machen:** Node → Updates-Tab mit der Liste der verfügbaren Pakete vor dem Upgrade – gutes Beweisbild für ein Wartungsprotokoll ("welche Pakete wurden an diesem Tag aktualisiert").

```bash
apt update
apt list --upgradable
apt dist-upgrade -y
```

### Schritt 5: Reboot bei Kernel-Update

```bash
ls /var/run/reboot-required 2>/dev/null && echo "Reboot nötig" || echo "Kein Reboot nötig"
uname -r
dpkg -l | grep pve-kernel | grep -v headers
reboot
```

### Schritt 6: Nach dem Reboot verifizieren

```bash
pveversion -v
systemctl --failed
pvecm status
```

### Schritt 7: Ceph-Flags zurücksetzen und Wartungsmodus beenden

```bash
ceph osd unset noout
ceph osd unset nobackfill
ceph osd unset norecover
ceph status

ha-manager crm-command node-maintenance disable pve2
```

### Schritt 8: Nächsten Node wiederholen

Erst wenn `pve2` vollständig verifiziert ist (Quorum, Ceph HEALTH_OK, HA-Status normal), mit `pve3` fortfahren.

---

## Major-Version-Upgrade (z. B. PVE 8 → 9)

Deutlich größerer Eingriff – zusätzliche Schritte gegenüber dem Routine-Update:

```bash
# Vorab auf JEDEM Node: Diagnose-Tool ausführen
pve8to9 --full
# Rote Einträge MUESSEN vor dem Upgrade behoben werden, gelbe Warnungen prüfen
```

> 📸 **Screenshot machen:** Die vollständige Terminal-Ausgabe von `pve8to9 --full` eines Nodes (oder als Textdatei archivieren) – das ist die wichtigste Dokumentation eines jeden Major-Upgrades und wird bei Rückfragen zum Wartungsprotokoll gebraucht.

Bei Ceph im Einsatz: **vor** dem PVE-Upgrade muss Ceph auf die neue Major-Version (z. B. Reef → Squid) aktualisiert sein. Reihenfolge ist kritisch:

```bash
ceph status
ceph osd set noout
ceph osd set nobackfill
ceph osd set norecover

# Auf jedem Node: Ceph-Repository auf die neue Codename-Version umstellen, dann
apt update && apt dist-upgrade -y

# Erst NACH allen Nodes:
ceph osd unset noout
ceph osd unset nobackfill
ceph osd unset norecover
ceph status
```

Wenn PBS im Einsatz ist: **PBS muss vor dem PVE-Upgrade aktualisiert sein**, nicht danach.

Nach dem Upgrade aller Nodes auf die neue Major-Version: Browser-Cache leeren/hart neu laden (Web-UI), `pve8to9`-Checkliste erneut prüfen, HA-Gruppen-Migration zu HA-Regeln (PVE 9) automatisch nach Abschluss aller Nodes.

---

## Vorsichtsmaßnahmen bei kritischen Clustern

```bash
# Vor dem Reboot: Cluster-Dienste kontrolliert stoppen, falls extra vorsichtig vorgegangen werden soll
systemctl stop pve-cluster
systemctl stop corosync
# (nur unmittelbar vor dem Reboot – trennt den Node bis zum Neustart vom Cluster)
```

```bash
# Snapshot der Root-Filesystem vor dem Upgrade (zusätzliches Sicherheitsnetz)
# ZFS:
zfs snapshot rpool/ROOT/pve-1@pre-upgrade-$(date +%Y%m%d)

# LVM (ext4-OS):
lvcreate -L 10G -s -n root-snap /dev/pve/root
```

> Bei einem 3-Node-Cluster dürfen niemals zwei Nodes gleichzeitig offline sein – das kostet das Quorum. Bei größeren Clustern gilt: immer Mehrheit der Votes online halten.

---

## Update-Benachrichtigungen automatisieren (ohne automatisch zu patchen)

```
Web-UI: Datacenter → Options → E-Mail-Benachrichtigung für verfügbare Updates aktivieren
```

> 📸 **Screenshot machen:** Datacenter → Options → die Zeile für Update-Benachrichtigungen mit aktivierter E-Mail-Option – kleiner, aber oft übersehener Schalter.

> Updates sollten **nicht** vollautomatisch eingespielt werden – nur die Benachrichtigung über verfügbare Updates automatisieren, die Durchführung bleibt ein geplanter, manueller Vorgang im Wartungsfenster.

---

## Best Practices

✅ Immer `apt dist-upgrade`, niemals `apt upgrade`
✅ Rolling Update: ein Node nach dem anderen, niemals parallel
✅ Bei Ceph: `noout`/`nobackfill`/`norecover` während der kurzen Downtime setzen, danach sofort zurücksetzen
✅ `pve8to9`-Diagnosetool (oder äquivalent der Zielversion) vor jedem Major-Upgrade auf jedem Node ausführen
✅ Alle Nodes eines Clusters am selben Tag/Wartungsfenster aktualisieren – keine gemischten Versionen über längere Zeit
✅ Reboot nach jedem Kernel-Update, auch wenn der neue Kernel schon zuvor opt-in liefen

⚠️ Bei HA-VMs während des Updates: Node-Wartungsmodus nutzen, damit der HA-Manager keine ungewollten Migrationen auslöst
⚠️ Veeam-Backup-Kompatibilität nach Major-Upgrades prüfen – neue QEMU-Versionen haben in der Vergangenheit zu Inkompatibilitäten geführt
⚠️ Niemals über die Proxmox-GUI-Konsole aktualisieren (Verbindungsabbruch während des Updates möglich) – SSH mit `tmux`/`screen` bevorzugen

❌ Alle Cluster-Nodes gleichzeitig aktualisieren
❌ Update ohne vorheriges VM-Backup
❌ Ceph-Upgrade nach dem PVE-Upgrade statt davor

---

## Häufige Fehler & Lösungen

| Problem | Ursache | Lösung |
|---|---|---|
| `apt dist-upgrade` schlägt mit 401 fehl | Enterprise-Repo aktiv ohne Subscription | No-Subscription-Repo aktivieren |
| APT will `proxmox-ve` deinstallieren | Gemischte Repository-Codenamen (z. B. Bookworm + Trixie) | Alle Repos auf denselben Codenamen prüfen |
| VM startet nach Migration auf Zielnode nicht | CPU-Typ nicht auf beiden Nodes unterstützt | CPU-Typ prüfen, ggf. `x86-64-v2-AES` setzen |
| Cluster verliert Quorum während Update | Zwei Nodes gleichzeitig offline | Reihenfolge strikt einhalten, nie parallel updaten |
| SSH-Verbindung bricht während Update ab | Netzwerk-Instabilität oder Reboot mitten im Vorgang | `tmux`/`screen` verwenden, Update fortsetzen nach Reconnect |

---

## Weiterführende Links

| Ressource | URL |
|---|---|
| Upgrade from 8 to 9 (offiziell) | https://pve.proxmox.com/wiki/Upgrade_from_8_to_9 |
