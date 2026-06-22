# Kapitel 6: Monitoring & Alerting

> **Version:** Proxmox VE 9.x · **Letzte Aktualisierung:** 2026-06  
> **Verwandte Kapitel:** [Kapitel 1](01-grundkonfiguration.md) (E-Mail-Setup) · [Kapitel 5](05-sicherheit-hardening.md) (API-Token)

---

## Wann anwenden?

> - Benachrichtigungen für Backup-Fehler, Hardware-Probleme, Ressourcenengpässe einrichten
> - Grafana-Dashboards für Langzeit-Performance-Monitoring
> - Vorhandene Systeme (Zabbix, Checkmk) integrieren

---

## Überblick

| Ebene | Tool | Stärke |
|-------|------|--------|
| Native Alerts | PVE Notification System | E-Mail/Gotify für Backups, SMART, Cluster |
| Metriken extern | InfluxDB + Grafana | Langzeit-Trends, visuelle Dashboards |
| Metriken extern | Prometheus + Grafana | Kubernetes-kompatibel, Pull-Modell |
| Enterprise | Zabbix / Checkmk | SNMP, Agenten, komplexe Alert-Regeln |

---

## 1. Native PVE Benachrichtigungen (PVE 9.x)

### 1.1 Notification Target anlegen

```
Datacenter → Notifications → Add

→ SMTP:
  Host:     smtp.office365.com
  Port:     587, Mode: TLS (STARTTLS)
  Username: alert@firma.de
  From:     proxmox@firma.de
  To:       admin@firma.de

→ Gotify (Mobile Push):
  Server:  https://gotify.firma.de
  Token:   [App-Token]
```

### 1.2 Notification Rules konfigurieren

```
Datacenter → Notifications → Notification Rules → Add

Regel 1 – Backup-Fehler:
  Matchers: severity >= error, source = backup
  Target:   SMTP + Gotify

Regel 2 – SMART-Fehler:
  Matchers: severity >= warning, source = system
  Target:   SMTP

Regel 3 – Cluster-Probleme:
  Matchers: severity >= error, source = cluster
  Target:   SMTP
```

---

## 2. InfluxDB + Grafana (empfohlen)

### 2.1 InfluxDB 2.x installieren

```bash
# Auf dedizierten Monitoring-Server:
apt install -y influxdb2
systemctl enable --now influxdb

# InfluxDB einrichten:
influx setup \
    --username admin \
    --password PASSWORT \
    --org firma \
    --bucket proxmox \
    --retention 30d \
    --force

# Write-Token für PVE generieren:
influx auth create \
    --org firma \
    --write-bucket proxmox \
    --description "Proxmox Write Token"
# Token notieren!
```

### 2.2 PVE Metriken-Export

```bash
# Web-UI: Datacenter → Metric Server → Add → InfluxDB

# Via API:
pvesh create /cluster/metrics/server/influxdb-main \
    --type influxdb \
    --server 192.168.1.60 \
    --port 8086 \
    --influxdbproto http \
    --organization firma \
    --bucket proxmox \
    --token INFLUXDB_TOKEN \
    --enable 1

# Metriken nach ~60s in InfluxDB prüfen:
# https://192.168.1.60:8086 → Data Explorer → proxmox Bucket
```

### 2.3 Grafana installieren

```bash
curl -fsSL https://packages.grafana.com/gpg.key \
    | gpg --dearmor > /etc/apt/trusted.gpg.d/grafana.gpg
echo "deb https://packages.grafana.com/oss/deb stable main" \
    > /etc/apt/sources.list.d/grafana.list
apt update && apt install -y grafana
systemctl enable --now grafana-server
# Web-UI: http://192.168.1.60:3000 (admin/admin)
```

```
Grafana: Configuration → Data Sources → Add → InfluxDB
  Query Language: Flux
  URL:            http://localhost:8086
  Organization:   firma
  Token:          [InfluxDB-Token]
  Default Bucket: proxmox

Dashboards → Import → ID: 21259
(Proxmox Cluster Summary – populäres Community-Dashboard)
```

---

## 3. Prometheus + Grafana (Alternative)

```bash
# PVE Prometheus Exporter (auf jedem PVE-Node):
apt install -y prometheus-pve-exporter

# /etc/pve-exporter.yaml:
cat > /etc/pve-exporter.yaml << 'EOF'
default:
  user: monitoring@pve
  token_name: monitoring-token
  token_value: TOKEN_SECRET
  verify_ssl: false
EOF

systemctl enable --now prometheus-pve-exporter
# Metriken unter: http://NODE-IP:9221/pve?target=localhost
```

```yaml
# prometheus.yml scrape config:
scrape_configs:
  - job_name: proxmox
    static_configs:
      - targets: [192.168.1.10, 192.168.1.11]
    metrics_path: /pve
    params:
      target: [localhost]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: NODE-IP:9221
```

---

## 4. Zabbix-Integration

```bash
# Zabbix-Agent 2 auf jedem PVE-Node:
apt install -y zabbix-agent2

# /etc/zabbix/zabbix_agent2.conf:
Server=192.168.1.70
ServerActive=192.168.1.70
Hostname=pve01.firma.local

systemctl enable --now zabbix-agent2
```

```
In Zabbix:
Configuration → Hosts → Create Host
Template: Linux by Zabbix agent 2
+ Community-Template "Proxmox VE by HTTP" importieren
https://www.zabbix.com/integrations/proxmox
```

---

## 5. Empfohlene Alert-Schwellwerte

| Alarm | Schwellwert | Priorität |
|-------|-------------|----------|
| CPU-Last | > 90 % für 15 Min | Warning |
| RAM-Auslastung | > 85 % | Warning |
| ZFS-Pool degraded | Status != ONLINE | Critical |
| Backup-Job fehlgeschlagen | Letzter Backup > 24h | Critical |
| SMART-Fehler | Reallocated Sectors > 0 | Critical |
| Cluster-Quorum verloren | Quorate = No | Critical |
| Node nicht erreichbar | Ping-Timeout 3 Min | Critical |
| PBS-Datastore > 85 % voll | | Warning |

---

## Best Practices

✅ Notification-System sofort nach Grundkonfiguration einrichten  
✅ InfluxDB-Retention: 30–90 Tage  
✅ Grafana-Zugriff mit LDAP/AD absichern  
✅ Backup-Verifizierungsfehler immer Critical-Priorität

⚠️ PVE schickt Metriken im 1-Minuten-Takt (kein Echtzeit-Monitoring)  
⚠️ prometheus-pve-exporter benötigt API-Token (nicht Root-Passwort)

---

## Häufige Fehler & Lösungen

**PVE-Metriken erscheinen nicht in InfluxDB**
```
Ursache: Falscher Token, falsche Bucket-ID oder InfluxDB nicht erreichbar.
Prüfen:  curl -I http://192.168.1.60:8086/health
         pvesh get /cluster/metrics/server/
Lösung:  Metric-Server-Konfig löschen und neu anlegen.
```

**E-Mail-Benachrichtigungen kommen nicht an**
```
Prüfen:  echo "Test" | mail -s "PVE Test" admin@firma.de
         mailq (hängende Mails)
         journalctl -u postfix
Lösung:  /etc/postfix/main.cf prüfen, postmap neu ausführen.
```

---

## Weiterführende Links

| Ressource | URL |
|-----------|-----|
| PVE Notifications | https://pve.proxmox.com/wiki/Notifications |
| External Metric Server | https://pve.proxmox.com/wiki/External_Metric_Server |
| Prometheus PVE Exporter | https://github.com/prometheus-pve/prometheus-pve-exporter |
| Zabbix Proxmox | https://www.zabbix.com/integrations/proxmox |
| Grafana Dashboard | https://grafana.com/grafana/dashboards/21259 |

---

**← Vorheriges Kapitel:** [Kapitel 5 – Sicherheit & Hardening](05-sicherheit-hardening.md)  
**← Zurück zur Übersicht:** [README.md](../README.md)
