# 07 – Backup mit Veeam Agent und getesteter Wiederherstellung

[← Zur Übersicht](../README.md) · [Troubleshooting-Index](../TROUBLESHOOTING.md)

> **Kurzfassung:** Der Homelab-Host wird als Vollsicherung inklusive aller VMs auf eine externe 1-TB-HDD gesichert. Die Sicherung ist verschlüsselt, startet automatisch beim Anstecken der Platte und meldet die Platte danach selbst ab (Offline-Kopie). Die Wiederherstellung wurde real getestet und per Prüfsumme belegt, das Recovery Media per Boot-Test.

---

## 1. Ausgangslage

Laut [Mini-Risikoanalyse](../SECURITY-COMPLIANCE.md#3-mini-risikoanalyse) war Datenverlust das höchste Risiko im Lab: Es gab nur VM-Snapshots, und die lagen auf demselben Datenträger wie die VMs. Ein Defekt der Systemplatte, ein voller Datenträger oder Ransomware hätten Snapshots und VMs gleichzeitig getroffen. Mit dem [Plattentausch](06-hardware-speichererweiterung.md) wurde die 1-TB-HDD als getrenntes Sicherungsziel frei.

## 2. Anforderungen

| Anforderung | Umsetzung |
|---|---|
| Getrenntes Medium | externe Seagate 1 TB (HDD) per USB, nicht dauerhaft angeschlossen |
| Schutz gegen Ransomware | Platte ist nur während der Sicherung verbunden und wird danach automatisch ausgeworfen |
| Vertraulichkeit | Sicherungsdateien AES-verschlüsselt (enthält u. a. die AD-Datenbank des DC) |
| Wiederherstellbarkeit des ganzen Hosts | Vollsicherung („Entire computer“), Bare-Metal-Restore per Recovery Media |
| Konsistenz | VMs vor der Sicherung sauber heruntergefahren (Client → Zammad → DC01) |
| Nachweis | Test-Restore mit Prüfsummenvergleich |

## 3. Konfiguration

Werkzeug: **Veeam Agent for Microsoft Windows (Free)**, installiert auf dem Host per `winget install Veeam.VeeamAgent`.

| Einstellung | Wert | Begründung |
|---|---|---|
| Job-Name | `m720q-Vollsicherung` | Gerät und Umfang auf einen Blick |
| Modus | Entire computer, **ohne** externe USB-Laufwerke | Host + VMs in einem Abbild; das Sicherungsziel darf sich nicht selbst sichern |
| Ziel | `D:\VeeamBackup\` (Seagate, Local storage) | getrenntes Medium |
| Aufbewahrung | 7 Tage, Tage ohne Sicherung zählen nicht | passt zur abziehbaren Platte: es bleiben immer 7 echte Wiederherstellungspunkte |
| Verschlüsselung | aktiviert | Passwort im Passwortmanager, nicht im Vault oder Repo |
| Zeitplan | **kein** täglicher Termin, Start „beim Anstecken des Ziels“, höchstens alle 12 Stunden | nachts ist die Platte nicht verbunden; ein fester Termin würde nur Fehlermeldungen erzeugen und ggf. laufende VMs absturzkonsistent sichern |
| Nach der Sicherung | Wechseldatenträger auswerfen | Offline-Kopie (Ransomware-Schutz) |

![Backup-Modus: Entire computer, ohne externe USB-Laufwerke](../screenshots/101-veeam-backup-mode-entire-computer.png)
![Ziel D:\VeeamBackup, 7 Tage Aufbewahrung](../screenshots/102-veeam-ziel-seagate-7-tage.png)
![Zeitplan: Start beim Anstecken, Auswerfen nach der Sicherung](../screenshots/103-veeam-zeitplan-beim-anstecken.png)
![Zusammenfassung: Verschlüsselung aktiviert](../screenshots/104-veeam-zusammenfassung-verschluesselt.png)
![Zusammenfassung: Zeitplan „At storage attach“](../screenshots/105-veeam-zusammenfassung-zeitplan.png)

## 4. Erste Vollsicherung

Die erste Sicherung lief rund eine Stunde und belegt etwa 181 GB (rund 230 GB belegt auf C:, Veeam komprimiert und lässt Auslagerungs- und temporäre Dateien weg). Auf der 1-TB-Platte bleibt genug Platz für die inkrementellen Folgesicherungen.

![Erste Vollsicherung erfolgreich](../screenshots/106-veeam-erste-vollsicherung-erfolgreich.png)
![Sicherungsdateien: Metadatei (.vbm) und Vollsicherung (.vbk)](../screenshots/107-veeam-sicherungsdateien.png)

**Stolperstein:** Beim ersten Anlegen des Jobs war die Verschlüsselung noch nicht aktiviert, und die Sicherung startete sofort. Der Job wurde abgebrochen und mit Verschlüsselung neu gestartet. Anschließend wurde geprüft, dass im Zielordner nur die verschlüsselte Sicherung liegt und kein unverschlüsseltes Fragment übrig geblieben ist. Verschlüsselung muss vor der ersten Sicherung feststehen; nachträglich aktiviert erzwingt sie eine neue Vollsicherung.

## 5. Test-Restore

Ein Backup gilt erst als Backup, wenn die Wiederherstellung nachweislich funktioniert. Getestet wurde die Wiederherstellung auf Dateiebene:

1. Veeam → *Restore Files* → Wiederherstellungspunkt vom 14.11.2026, Entschlüsselung per Passwort
2. Datei `zammad.txt` (116 Byte, vor der Sicherung angelegt) aus dem Backup an einen anderen Ort kopiert, um nichts zu überschreiben
3. Original und Wiederherstellung per SHA-256 verglichen

![Backup-Browser mit dem Inhalt der verschlüsselten Sicherung](../screenshots/108-veeam-restore-dateibrowser.png)

```powershell
Get-FileHash "$env:USERPROFILE\Desktop\zammad.txt", "$env:USERPROFILE\Desktop\RESTORED-zammad.txt"
```

![Identische SHA-256-Prüfsummen](../screenshots/109-restore-test-hash-identisch.png)

**Ergebnis:** Beide Prüfsummen sind identisch, die Datei wurde bitgenau aus der verschlüsselten Sicherung wiederhergestellt.

## 6. Recovery Media

Für den Fall, dass Windows nicht mehr startet, wurde am 14.11.2026 ein bootfähiger USB-Stick (8 GB) mit der Veeam Recovery Environment erstellt (*Create Recovery Media*).

| Option | Wert | Begründung |
|---|---|---|
| Ziel | USB-Stick (wird vollständig formatiert) | sofort startfähig, kein Umweg über ISO; die Seagate (D:) bewusst **nicht** gewählt |
| Netzwerkeinstellungen dieses Computers | aktiviert | Zugriff auf Sicherungen auch über das Netzwerk |
| Hardwaretreiber dieses Computers | aktiviert | Recovery-Umgebung erkennt NVMe und Netzwerk des M720q |
| Entschlüsselungsschlüssel auf dem Medium | **deaktiviert** | Stick liegt neben der Sicherungsplatte; mit Schlüssel wäre die Verschlüsselung bei Verlust beider wertlos. Passwort liegt im Passwortmanager |

Danach verschwand die Warnung „Recovery media not created“ im Veeam-Dashboard. Zum Zeitpunkt der Aufnahme existieren drei Läufe: der abgebrochene unverschlüsselte Erstversuch (grau), die Vollsicherung (~181 GB) und eine manuell gestartete inkrementelle Testsicherung (~5 GB). Im Zielordner liegen nur Metadatei (`.vbm`), Vollsicherung (`.vbk`) und Inkrement (`.vib`) – keine Reste des Erstversuchs.

![Recovery Media: Ziel USB-Stick, Treiber und Netzwerk, ohne Schlüssel](../screenshots/110-veeam-recovery-media-optionen.png)
![Zusammenfassung vor dem Erstellen](../screenshots/111-veeam-recovery-media-zusammenfassung.png)
![Recovery Media erfolgreich erstellt](../screenshots/112-veeam-recovery-media-erstellt.png)
![Dashboard ohne Warnung: abgebrochener Erstlauf, Vollsicherung, Inkrement](../screenshots/113-veeam-dashboard-ohne-warnung.png)
![Zielordner: nur .vbm, .vbk und .vib](../screenshots/114-veeam-zielordner-kette.png)

**Status:** erstellt und am 17.11.2026 per Boot-Test belegt (siehe 6.1).

### 6.1 Boot-Test (17.11.2026)

Ziel: Nachweisen, dass das Medium im Ernstfall startet, die Sicherung findet und sie ohne gespeicherten Schlüssel entschlüsseln kann. **Ausdrücklich ohne Restore.**

Ablauf: VMs heruntergefahren (Client → Zammad → DC01), Seagate angesteckt, Neustart, F12-Bootmenü, vom Recovery-USB gestartet.

| Prüfpunkt | Ergebnis |
|---|---|
| Recovery-Umgebung startet (Secure Boot, Treiber) | ✅ Veeam Recovery Media 13.1.2 auf Basis Windows 11 25H2 |
| Sicherung auf der Seagate wird gefunden | ✅ automatisch erkannt: `D:\VeeamBackup\m720q-Vollsicherung\m720q-Vollsicherung.vbm`, Windows 11 Pro, Volume C |
| Entschlüsselung ohne Schlüssel auf dem Medium | ✅ Passwortabfrage erscheint, Entschlüsselung mit dem Passwort aus dem Passwortmanager |
| Wiederherstellungspunkte lesbar | ✅ Vollsicherung (14.11.2026, 10:47) und Inkrement (14.11.2026, 12:47), Disk 0 |

Danach im Assistenten vor „Restore Mode“ abgebrochen, normal gestartet, VMs wieder hochgefahren (DC01 → Zammad → Client), Platte und Stick abgezogen und getrennt vom Host aufbewahrt.

![Startmenü der Veeam Recovery Media 13.1.2](../screenshots/115-veeam-recovery-media-hauptmenue.jpg)
![Bare Metal Recovery: Sicherung auf der Seagate automatisch gefunden](../screenshots/117-veeam-bmr-sicherung-gefunden.jpg)
![Nach Passworteingabe: Wiederherstellungspunkte lesbar](../screenshots/118-veeam-bmr-restore-points-entschluesselt.jpg)

**Stolperstein:** Der erste Versuch über *Windows Recovery Environment* meldete „Auf dem Computer wurde kein Systemabbild gefunden“. Das ist kein Fehler der Sicherung: Dieser Menüpunkt startet die Windows-eigene Systemabbild-Wiederherstellung und kennt nur Windows-Systemabbilder, keine Veeam-Sicherungen. Der richtige Weg ist *Bare Metal Recovery*. Für den Ernstfall im Betriebsablauf festgehalten.

![Falscher Menüpunkt: Windows RE findet kein Systemabbild](../screenshots/116-windows-re-kein-systemabbild.jpg)

## 7. Betrieb

- **Ablauf:** VMs herunterfahren (Client → Zammad → DC01), Platte anstecken, Sicherung läuft automatisch, Platte wird ausgeworfen, abziehen, VMs wieder starten (DC01 → Zammad → Client).
- **Rhythmus:** wöchentlich und vor jedem größeren Umbau (z. B. vor der L2-Delegation im AD-Rollenmodell).
- **Beobachten:** SMART-Wert C7 (CRC-Fehler) der HDD; steigt er nach Sicherungen deutlich, liegt es an Gehäuse oder Kabel.
- **Ernstfall:** vom Recovery-USB starten (F12) → *Bare Metal Recovery* (nicht *Windows Recovery Environment*) → Local storage → Passwort aus dem Passwortmanager.
- **Zusätzliche Rückfallebene:** Die alte Systemplatte (WD 256 GB) liegt als startfähiger Klon vom 14.11.2026 offline.

## 8. Offene Punkte

| Punkt | Warum |
|---|---|
| Bare-Metal-Restore einmal testen (z. B. auf die freie WD) | Datei-Restore belegt die Lesbarkeit, nicht die Wiederherstellung des ganzen Hosts |
| Anwendungskonsistente Sicherung des DC (Windows Server Backup / System State in der VM) | die Host-Sicherung erfasst laufende VMs nur absturzkonsistent; deshalb werden sie vorher heruntergefahren |
| Zweite Kopie außer Haus (3-2-1-Regel) | aktuell zwei Kopien (Original + HDD) an einem Ort |

[← Zur Übersicht](../README.md) · [Troubleshooting-Index](../TROUBLESHOOTING.md)
