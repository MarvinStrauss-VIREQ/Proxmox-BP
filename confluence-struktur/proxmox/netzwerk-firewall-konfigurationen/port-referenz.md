<!-- Space: KNOWLEDGEBASE -->
<!-- Parent: Proxmox - Netzwerk- & Firewall-Konfigurationen -->
<!-- Title: Proxmox - Port-Referenz -->
<!-- Label: pve-netzwerk, pve-sicherheit, referenz -->

# Port-Referenz

> **PVE 9.x (Debian 13 Trixie)** · Letzte Aktualisierung: 2026-06
> Quelle: adaptiert aus Kapitel 07 – Firewall & IPsets. **Vor Sync gegen bestehenden Confluence-Stand prüfen.**
> Verwandte Seiten: [Firewall](firewall.md) · [Ceph Cluster - Netzwerk](ceph-cluster-netzwerk.md)

---

## Wann anwenden?

- Schnellreferenz beim Erstellen oder Audit von Firewall-Regeln
- Beim Troubleshooting von Konnektivitätsproblemen zwischen Nodes, Storage-Netz oder Admin-Workstations

---

## Port-Tabelle (3-Node-Ceph-Referenz)

| Port(s) | Protokoll | Dienst | Quelle (IPset) | Interface | Pflicht |
|---|---|---|---|---|:---:|
| 22 | TCP | SSH | `mgmt-hosts`, `cluster-mgmt` | vmbr0 | ✅ |
| 8006 | TCP | Proxmox Web UI | `mgmt-hosts` | vmbr0 | ✅ |
| 5405 | UDP | Corosync Ring 0 | `corosync-ring0` | ens19 | ✅ |
| 5406 | UDP | Corosync Ring 1 | `cluster-mgmt` | vmbr0 | ✅ |
| 60000–60050 | TCP | VM/CT Live-Migration | `cluster-mgmt` | vmbr0 | ✅ |
| 5900–5999 | TCP | VNC / noVNC-Konsole | `mgmt-hosts` | vmbr0 | ✅ |
| 3128 | TCP | SPICE-Proxy | `mgmt-hosts` | vmbr0 | ✅ |
| 3300 | TCP | Ceph Monitor v2 | `ceph-net` | bond1 | ✅ (Ceph) |
| 6789 | TCP | Ceph Monitor v1 | `ceph-net` | bond1 | ✅ (Ceph) |
| 6800–7300 | TCP | Ceph OSD / MDS / MGR | `ceph-net` | bond1 | ✅ (Ceph) |
| 5403 | TCP | Corosync QDevice (QNetd) | `pbs-qdevice` | vmbr0 | ✅ (2-Node) |
| 8007 | TCP | Proxmox Backup Server API | `mgmt-hosts`/`cluster-mgmt` | vmbr0 | ✅ (PBS) |
| 8443 | TCP | Proxmox Datacenter Manager Web UI | `mgmt-hosts` | vmbr0 | Optional |
| — | ICMP | Ping / Monitoring | alle IPsets | alle | ✅ |
| 9100 | TCP | Prometheus Node Exporter | `mgmt-hosts` | vmbr0 | Optional |
| 8443 | TCP | Ceph Dashboard (direkt) | `mgmt-hosts` | vmbr0 | Optional |
| 4789 | UDP | VXLAN (SDN) | `cluster-mgmt` | vmbr0 | Optional |

---

## pve-firewall CLI-Referenz

| Befehl | Funktion |
|---|---|
| `pve-firewall compile` | Regeln kompilieren + Syntaxprüfung (kein Output = OK) |
| `pve-firewall status` | Firewall-Status anzeigen |
| `pve-firewall restart` | Service neu starten |
| `pve-firewall localnet` | Lokale Netzwerk-Info anzeigen |
| `pve-firewall simulate` | Regel simulieren **ohne** Aktivierung |
| `pve-firewall stop` | ⚠️ Entfernt **alle** Regeln – Node ungeschützt! |

### `simulate` – Regeln testen ohne Aktivierung

Das wichtigste Debugging-Tool: simuliert, ob eine bestimmte Verbindung erlaubt oder geblockt würde – ohne die Firewall zu verändern. Ausgabe `ACCEPT` oder `DROP`.

```bash
pve-firewall simulate \
  --from outside \        # Quell-Zone (outside = außerhalb des Clusters)
  --to host \             # Ziel-Zone (host = dieser Node)
  --source <quell-ip> \   # Quell-IP-Adresse
  --dport <port> \        # Zielport
  --protocol tcp|udp      # Protokoll (kein icmp-Support im simulate-Befehl)

# SSH von Admin erlaubt?
pve-firewall simulate --from outside --to host \
  --source 172.30.7.50 --dport 22 --protocol tcp

# Ceph Monitor von Storage-Netz erlaubt?
pve-firewall simulate --from outside --to host \
  --source 10.20.20.12 --dport 3300 --protocol tcp

# Verbose-Ausgabe (zeigt, welche Regel greift)
pve-firewall simulate --from outside --to host \
  --source 172.30.7.50 --dport 22 --protocol tcp --verbose 1
```

### Weitere Debugging-Befehle

```bash
pve-firewall compile                         # Compile-Fehler anzeigen
journalctl -u pve-firewall --since "10 minutes ago" | tail -50
iptables -L INPUT -n -v --line-numbers
ipset list cluster-mgmt
ipset list ceph-net

# Connectivity-Tests
nc -vzu 10.10.20.12 5405 && echo "Ring 0: OK" || echo "Ring 0: BLOCKED"
nc -vz  10.20.20.12 3300 && echo "Ceph MON: OK" || echo "Ceph MON: BLOCKED"
```

---

## Weiterführende Links

| Ressource | URL |
|---|---|
| Proxmox VE Firewall | https://pve.proxmox.com/wiki/Firewall |
| Ceph Network Config Reference | https://docs.ceph.com/en/squid/rados/configuration/network-config-ref/ |
