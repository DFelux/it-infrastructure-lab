# Netzwerk- und Security-Erweiterung (Roadmap)

[← Zur Übersicht](../README.md) · [Troubleshooting-Index](../TROUBLESHOOTING.md)

## 🔐 Netzwerk- und Security-Erweiterung (in Arbeit)

Ausbau des Labs um typische Komponenten einer abgesicherten 
Unternehmensumgebung. Status: **teilweise umgesetzt**, einzelne Bausteine werden nach 
Abschluss mit Konfiguration, Screenshots und Troubleshooting dokumentiert. 
Zammad ist inzwischen per TLS über eine interne CA abgesichert, und die Ticketsystem-VM 
filtert eingehenden Verkehr per Host-Firewall (ufw); eine zentrale 
Firewall im Netz fehlt noch, genau das soll der weitere Ausbau ändern.

| Baustein | Ziel | Status |
|---|---|---|
| Virtuelle Firewall (OPNsense/pfSense) | Zentrale Absicherung des internen Netzes, Regelwerk, NAT/PAT, Portweiterleitung | ⬜ geplant |
| Interne CA und TLS-Zertifikate | X.509-Zertifikate, TLS für Zammad über Reverse Proxy (nginx) | ✅ umgesetzt, siehe [Meilenstein](03-pki-tls.md#meilenstein-interne-ca-und-tls-für-zammad) |
| Host-Firewall (ufw) | Eingehenden Verkehr auf der Ticketsystem-VM auf SSH und HTTPS beschränken | ✅ umgesetzt, siehe [Meilenstein](04-host-firewall.md#meilenstein-host-firewall-ufw-auf-der-ticketsystem-vm) |
| Compliance-Mapping (NIS2 / ISO 27001) | Maßnahmen den Anforderungen zuordnen, Lücken und Risiken bewerten | ✅ umgesetzt, siehe [SECURITY-COMPLIANCE.md](../SECURITY-COMPLIANCE.md) |
| VLAN-Segmentierung | Trennung von Server-, Client- und Management-Netz | ⬜ geplant |
| Netzwerkanalyse | Paketmitschnitte mit tcpdump/Wireshark, DNS-Prüfung mit dig | ⬜ geplant |
| Backup und Restore | Sicherung der VMs und Konfigurationen (inkl. Firewall) auf ein getrenntes Medium, Wiederherstellung testen | ⬜ geplant |

### Lernziele
- Firewall-Regeln planen, testen und anhand der Logs nachvollziehen
- Aufbau und Betrieb einer einfachen CA-Struktur, Ausstellen und Einbinden von Zertifikaten
- Fehlersuche auf Netzwerk- und Zertifikatsebene
- Technische Maßnahmen Sicherheitsanforderungen (NIS2, ISO 27001) zuordnen und Restrisiken bewerten

### Dokumentation je Baustein
Ziel · Aufbau (Netzwerkdiagramm) · Konfiguration · Tests · Troubleshooting · Screenshots


[← Zur Übersicht](../README.md) · [Troubleshooting-Index](../TROUBLESHOOTING.md)
