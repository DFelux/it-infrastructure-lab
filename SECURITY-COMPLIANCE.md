## Security Controls & Compliance-Mapping

> **Einordnung:** Dieses Homelab fällt selbstverständlich nicht unter NIS2. Der Abschnitt ist eine **Übung**: Die umgesetzten technischen Maßnahmen werden den Risikomanagement-Maßnahmen nach **NIS2 Art. 21 Abs. 2** und den Controls aus **ISO/IEC 27001:2022 Anhang A** gegenübergestellt. Ziel ist, Technik in Sicherheits- und Governance-Logik zu übersetzen und Lücken offen zu benennen. Es handelt sich nicht um einen Compliance-Nachweis.

### 1. Umgesetzte Maßnahmen

| Maßnahme im Lab | Nachweis | NIS2 Art. 21 (2) | ISO 27001 Anhang A |
|---|---|---|---|
| Interne Zertifizierungsstelle, Root-CA per GPO an Domänen-Clients verteilt; Zammad nur noch per HTTPS (nginx-Reverse-Proxy, Port 443) | Zertifikat mit SAN und Server-Auth-EKU, Screenshots im Repo | h) Kryptografie und Verschlüsselung | 8.24 Einsatz von Kryptografie |
| Anwendungsport an `127.0.0.1` gebunden, nur über den Proxy erreichbar | Negativtest: Port 8080 aus dem Host-only-Netz nicht erreichbar, 443 erreichbar | e) Sicherheit bei Wartung von Systemen | 8.20 Netzwerksicherheit, 8.9 Konfigurationsmanagement |
| Host-Firewall (ufw): Default Deny eingehend; SSH nur aus dem Management-Netz und vom Admin-Notebook | ufw-Regelwerk, Journal-Eintrag zum SSH-Login, Test-NetConnection | i) Zugriffskontrolle; g) Cyberhygiene | 8.20 Netzwerksicherheit, 8.22 Trennung von Netzwerken |
| Active Directory mit OU-Struktur; Rechtevergabe über Gruppen und Rollen | AD-Struktur, Screenshots | i) Zugriffskontrolle | 5.15 Zugangssteuerung, 5.18 Zugangsrechte |
| Azure RBAC: integrierte Rolle „Reader“ und Custom Role „VM Restart Operator“ (Least Privilege) | Rollendefinition (JSON) im Repo `hybrid-identity-lab` | i) Zugriffskontrolle | 5.15 Zugangssteuerung, 8.2 Privilegierte Zugangsrechte |
| Hybride Identität: Entra Connect Sync (Password Hash Sync), Entra Hybrid Join | Sync-Status, Geräteansicht im Entra-Portal | i) Zugriffskontrolle | 5.16 Identitätsmanagement |
| MFA über Security Defaults | Konfiguration im Entra-Portal | j) Multi-Faktor-Authentifizierung | 8.5 Sichere Authentisierung |
| Tags und Resource Locks auf Ressourcengruppen | Governance-Konzept im Repo | i) Management von Anlagen | 5.9 Inventar der Informationswerte, 8.32 Änderungssteuerung |
| Ticketsystem (Zammad) für Störungen und Anfragen | Laufende Instanz, Doku im Repo | b) Bewältigung von Sicherheitsvorfällen | 5.24 Planung des Vorfallmanagements, 5.26 Reaktion auf Vorfälle |
| Jede Konfiguration wird mit Test und Screenshot belegt; Troubleshooting ist dokumentiert | README-Historie, Troubleshooting-Abschnitte | f) Bewertung der Wirksamkeit | Kap. 9.1 Überwachung und Messung; 5.37 Dokumentierte Betriebsabläufe |

### 2. Bekannte Lücken

Bewusst offen benannt. Ein Mapping ohne Lücken wäre bei einem Lab unglaubwürdig.

| Lücke | NIS2 Art. 21 (2) | ISO 27001 Anhang A | Mögliche nächste Maßnahme |
|---|---|---|---|
| Kein Backup- und Wiederherstellungskonzept (DC, Zammad-Datenbank) | c) Backup-Management, Wiederherstellung | 8.13 Sicherung von Informationen | Backup-Job einrichten und Wiederherstellung einmal real testen |
| Keine zentrale Protokollierung und kein Monitoring | b) Vorfallbewältigung; f) Wirksamkeit | 8.15 Protokollierung, 8.16 Überwachung | Logs zentral sammeln (z. B. Azure Monitor / Log Analytics) |
| Patch- und Schwachstellenmanagement nicht dokumentiert | e) Umgang mit Schwachstellen | 8.8 Management technischer Schwachstellen | Update-Routine festlegen und protokollieren |
| Docker-Images aus öffentlicher Registry ohne Prüfung | d) Sicherheit der Lieferkette | 5.21 Informationssicherheit in der IKT-Lieferkette | Versionen fixieren, Quellen und Update-Pfad dokumentieren |
| Kein Conditional Access (benötigt Entra ID P1) | j) MFA / kontinuierliche Authentifizierung | 8.5 Sichere Authentisierung | Konzept schriftlich ausarbeiten, ohne Lizenz nicht umsetzbar |
| Lab-typisch generische Konten und Kennwortrichtlinie nicht formalisiert | i) Zugriffskontrolle; g) Cyberhygiene | 5.17 Authentisierungsinformationen | Namenskonvention und Kennwortrichtlinie festlegen |

### 3. Mini-Risikoanalyse

Bewertung: Eintrittswahrscheinlichkeit (W) und Auswirkung (A), jeweils 1 = gering bis 3 = hoch. Risiko = W × A.

| Asset | Bedrohung | W | A | Risiko | Bestehende Maßnahme | Restrisiko / Behandlung |
|---|---|---|---|---|---|---|
| DC01 (AD DS) | Kompromittierung privilegierter Konten | 2 | 3 | 6 | MFA in der Cloud, Rollentrennung, Netztrennung | Mittel. Tiering für Admin-Konten und Protokollierung ergänzen |
| Zammad-Server | Unbefugter Zugriff auf Weboberfläche oder SSH | 2 | 2 | 4 | TLS, Default-Deny-Firewall, SSH nur aus dem Management-Netz | Gering bis mittel. Patch-Routine ergänzen |
| Lab-Daten (AD, Tickets) | Datenverlust durch Hardwaredefekt oder Fehlkonfiguration | 2 | 3 | 6 | Keine | **Hoch, unbehandelt.** Backup hat höchste Priorität |

### 4. Ausblick

Neben der technischen Administration interessiert mich die Frage, wie Sicherheitsmaßnahmen nachweisbar und steuerbar werden. Als nächste Schritte sind geplant, die Lücken oben schrittweise zu schließen und das Mapping zu einer kleinen Erklärung zur Anwendbarkeit (Statement of Applicability) auszubauen.
