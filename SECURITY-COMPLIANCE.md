# Security Controls & Compliance-Mapping

> **Einordnung:** Dieses Homelab fällt selbstverständlich nicht unter NIS2. Diese Übersicht ist eine **Übung**: Die umgesetzten technischen Maßnahmen werden den Risikomanagement-Maßnahmen nach **NIS2 Art. 21 Abs. 2** und den Controls aus **ISO/IEC 27001:2022 Anhang A** gegenübergestellt. Ziel ist, Technik in Sicherheits- und Governance-Logik zu übersetzen und Lücken offen zu benennen. Es handelt sich nicht um einen Compliance-Nachweis.
>
> Jeder Eintrag verweist auf den zugehörigen Beleg in der [README](README.md) bzw. im Ordner [`screenshots`](screenshots/). Maßnahmen aus der Azure-Umgebung sind im Repo [hybrid-identity-lab](https://github.com/DFelux/hybrid-identity-lab) dokumentiert.

## 1. Umgesetzte Maßnahmen

| Maßnahme im Lab | Nachweis | NIS2 Art. 21 (2) | ISO 27001 Anhang A |
|---|---|---|---|
| Eigene Root-CA (Key passphrasengeschützt), Serverzertifikat mit SAN und Server-Auth-EKU, Zammad nur noch per HTTPS über nginx (TLS 1.2/1.3) | [Meilenstein CA/TLS](README.md#meilenstein-interne-ca-und-tls-für-zammad) · [Zertifikatsprüfung](screenshots/42-openssl-verify-san.png) · [nginx auf 443](screenshots/43-nginx-ss-443.png) | h) Kryptografie und Verschlüsselung | 8.24 Einsatz von Kryptografie |
| Root-CA per GPO an Domänen-Clients verteilt; Integrität der Zertifikatsdatei per SHA-256-Hashvergleich geprüft | [GPO-Import](screenshots/46-gpo-root-ca-import.png) · [Verbindung sicher](screenshots/47-edge-verbindung-sicher.png) · [Zertifikatskette](screenshots/48b-zertifikat-hierarchie.png) | h) Kryptografie | 8.24 Einsatz von Kryptografie, 8.9 Konfigurationsmanagement |
| Gegenprobe: Rechner ohne Root-CA erhält eine Zertifikatswarnung, die Vertrauenskette wird also tatsächlich geprüft | [ERR_CERT_AUTHORITY_INVALID](screenshots/57-edge-cert-authority-invalid.png) | f) Bewertung der Wirksamkeit | 8.24 Einsatz von Kryptografie |
| Anwendungsport an `127.0.0.1` gebunden, Zammad nur über den Reverse Proxy erreichbar; Originalkonfiguration als `.bak` gesichert | [Port-Bindung](screenshots/44-zammad-port-localhost.png) · [curl HTTPS 200](screenshots/45-curl-https-200.png) | e) Sicherheit bei Wartung von Systemen | 8.20 Netzwerksicherheit, 8.9 Konfigurationsmanagement, 8.32 Änderungssteuerung |
| Host-Firewall (ufw): Default Deny eingehend; HTTPS offen, SSH nur aus dem Lab-Netz und vom Admin-Notebook; Quelladresse aus dem Log abgeleitet statt angenommen | [Regelwerk](README.md#regelwerk) · [Fehlersuche](README.md#fehlersuche-drei-stolpersteine) · [SSH-Journal](screenshots/59-ssh-journal-quelladresse-notebook.png) | i) Zugriffskontrolle; g) Cyberhygiene | 8.20 Netzwerksicherheit, 8.22 Trennung von Netzwerken |
| Wirksamkeitstests der Firewall: 443 erreichbar, 8080 nicht, SSH nur von erlaubten Quellen | [Verifikation](README.md#verifikation) · [ufw-Status](screenshots/60-ufw-status-final-und-ssh-journal.png) · [Port-Tests](screenshots/61-test-netconnection-8080-443.png) | f) Bewertung der Wirksamkeit | Kap. 9.1 Überwachung und Messung; 8.20 Netzwerksicherheit |
| SSH-Host-Key-Warnung nicht weggeklickt, sondern Fingerabdruck vor dem Entfernen des alten Eintrags verglichen (Ausschluss Man-in-the-Middle) | [Host-Key-Warnung](screenshots/55-ssh-hostkey-changed.png) | h) Kryptografie; j) gesicherte Kommunikation | 8.20 Netzwerksicherheit |
| Active Directory mit OU-Struktur (IT, Support, Vertrieb); Client in die Domäne aufgenommen | [Umgesetzter Stand](README.md#umgesetzter-stand) · [Domänenbeitritt](README.md#meilenstein-client-erfolgreich-in-domäne-aufgenommen) | i) Zugriffskontrolle, Management von Anlagen | 5.15 Zugangssteuerung, 5.16 Identitätsmanagement |
| Azure RBAC: integrierte Rolle „Reader“ und Custom Role „VM Restart Operator“ (Least Privilege) | [hybrid-identity-lab](https://github.com/DFelux/hybrid-identity-lab) | i) Zugriffskontrolle | 5.15 Zugangssteuerung, 8.2 Privilegierte Zugangsrechte |
| Hybride Identität: Entra Connect Sync (Password Hash Sync), Entra Hybrid Join | [hybrid-identity-lab](https://github.com/DFelux/hybrid-identity-lab) | i) Zugriffskontrolle | 5.16 Identitätsmanagement |
| MFA über Security Defaults | [hybrid-identity-lab](https://github.com/DFelux/hybrid-identity-lab) | j) Multi-Faktor-Authentifizierung | 8.5 Sichere Authentisierung |
| Tags und Resource Locks auf Ressourcengruppen | [hybrid-identity-lab](https://github.com/DFelux/hybrid-identity-lab) | i) Management von Anlagen | 5.9 Inventar der Informationswerte, 8.32 Änderungssteuerung |
| Ticketsystem (Zammad) mit Rollen Support L1–L3 als Grundlage einer Eskalationsstruktur | [Meilenstein Zammad](README.md#meilenstein-zammad-läuft) · [Rollen](screenshots/36-zammad-rollen.png) | b) Bewältigung von Sicherheitsvorfällen | 5.24 Planung des Vorfallmanagements, 5.26 Reaktion auf Vorfälle |
| Jede Störung als Troubleshooting dokumentiert, jeweils mit Symptom, Ursache, Lösung und Erkenntnis | Troubleshooting-Abschnitte in der [README](README.md) | b) Vorfallbewältigung; f) Wirksamkeit | 5.27 Erkenntnisse aus Informationssicherheitsvorfällen, 5.37 Dokumentierte Betriebsabläufe |
| Keine Schlüssel oder Zugangsdaten im Repo; `.gitignore` schließt `*.key`, `*.srl` und `*.csr` aus; Platzhalter statt Benutzernamen und Passwörtern | [Sicherheitsbetrachtung](README.md#sicherheitsbetrachtung) | g) Cyberhygiene | 5.17 Authentisierungsinformationen, 8.12 Verhinderung von Datenlecks |

## 2. Bekannte Lücken und bewusst akzeptierte Abweichungen

Bewusst offen benannt. Ein Mapping ohne Lücken wäre bei einem Lab unglaubwürdig.

| Lücke | NIS2 Art. 21 (2) | ISO 27001 Anhang A | Mögliche nächste Maßnahme |
|---|---|---|---|
| Kein Backup im eigentlichen Sinn: Es gibt [VM-Snapshots](screenshots/01-vm-snapshot-baum.png), diese liegen aber auf demselben Datenträger wie die VMs | c) Backup-Management, Wiederherstellung | 8.13 Sicherung von Informationen | Sicherung auf ein getrenntes Medium, Wiederherstellung einmal real testen |
| Keine Kapazitätsüberwachung: Das Host-Laufwerk lief unbemerkt voll und pausierte die VM ([VERR_DISK_FULL](screenshots/50-virtualbox-verr-disk-full.png)) | c) Aufrechterhaltung des Betriebs | 8.6 Kapazitätssteuerung | Speicherwarnung einrichten, verwaiste VM-Dateien regelmäßig prüfen |
| Keine zentrale Protokollierung und kein Monitoring | b) Vorfallbewältigung; f) Wirksamkeit | 8.15 Protokollierung, 8.16 Überwachung | Logs zentral sammeln (z. B. Azure Monitor / Log Analytics) |
| Keine zentrale Firewall im Netz, ufw schützt nur eine VM | i) Zugriffskontrolle | 8.20 Netzwerksicherheit, 8.22 Trennung von Netzwerken | Virtuelle Firewall (OPNsense) und VLAN-Segmentierung, siehe [Erweiterung](README.md#-netzwerk--und-security-erweiterung-in-arbeit) |
| SSH mit Passwort statt Schlüssel | j) Authentifizierung | 8.5 Sichere Authentisierung | Schlüsselauthentifizierung, Passwort-Login deaktivieren, siehe [Nächste Schritte](README.md#nächste-schritte) |
| Patch- und Schwachstellenmanagement nicht dokumentiert | e) Umgang mit Schwachstellen | 8.8 Management technischer Schwachstellen | Update-Routine festlegen und protokollieren |
| Docker-Images aus öffentlicher Registry ohne gesonderte Prüfung | d) Sicherheit der Lieferkette | 5.21 Informationssicherheit in der IKT-Lieferkette | Versionen fixieren, Quellen und Update-Pfad dokumentieren |
| Zertifikatserneuerung nicht automatisiert (Ablauf 19.10.2027) | h) Kryptografie | 8.24 Einsatz von Kryptografie | Erneuerung als wiederkehrende Aufgabe planen bzw. automatisieren |
| Zammad-Gruppen und Berechtigungen der Rollen L1–L3 noch nicht konfiguriert | b) Vorfallbewältigung; i) Zugriffskontrolle | 5.15 Zugangssteuerung, 5.24 Planung des Vorfallmanagements | Gruppen anlegen, Eskalation L1 → L2 durchspielen |
| Kein Conditional Access (benötigt Entra ID P1) | j) Multi-Faktor-Authentifizierung | 8.5 Sichere Authentisierung | Konzept schriftlich ausarbeiten, ohne Lizenz nicht umsetzbar |
| **Akzeptiert:** Root-CA-Key liegt auf demselben Server statt offline mit Zwischen-CA | h) Kryptografie | 8.24 Einsatz von Kryptografie | Im Lab bewusst vereinfacht, siehe [Sicherheitsbetrachtung](README.md#sicherheitsbetrachtung) |
| **Akzeptiert:** Mitgliedschaft in der Gruppe `docker` entspricht praktisch Root-Rechten | i) Zugriffskontrolle | 8.2 Privilegierte Zugangsrechte | Im Lab bewusster Kompromiss; im Betrieb rootless Docker oder sudo mit Protokollierung |

## 3. Mini-Risikoanalyse

Bewertung: Eintrittswahrscheinlichkeit (W) und Auswirkung (A), jeweils 1 = gering bis 3 = hoch. Risiko = W × A.

| Asset | Bedrohung | W | A | Risiko | Bestehende Maßnahme | Restrisiko / Behandlung |
|---|---|---|---|---|---|---|
| DC01 (AD DS) | Kompromittierung privilegierter Konten | 2 | 3 | 6 | MFA in der Cloud, Rollentrennung, Host-only-Netz | Mittel. Tiering für Admin-Konten und Protokollierung ergänzen |
| Zammad-Server | Unbefugter Zugriff auf Weboberfläche oder SSH | 2 | 2 | 4 | TLS, Port-Bindung an localhost, Default-Deny-Firewall, SSH nur von definierten Quellen | Gering bis mittel. Schlüsselauthentifizierung und Patch-Routine ergänzen |
| Lab-Daten (AD, Tickets) | Datenverlust durch Hardwaredefekt, vollen Datenträger oder Fehlkonfiguration | 2 | 3 | 6 | Nur Snapshots auf demselben Datenträger | **Hoch, faktisch unbehandelt.** Getrenntes Backup hat höchste Priorität |

## 4. Ausblick

Neben der technischen Administration interessiert mich die Frage, wie Sicherheitsmaßnahmen nachweisbar und steuerbar werden. Als nächste Schritte sind geplant, die Lücken oben schrittweise zu schließen und das Mapping zu einer kleinen Erklärung zur Anwendbarkeit (Statement of Applicability) auszubauen.
