# 06 – Hardware: Speicherengpass und Erweiterung

> **Kurzfassung:** Der Homelab-Host lief mit 256 GB Systemplatte voll. Nach Bestandsaufnahme und Variantenvergleich wurde eine Lösung gewählt, die **fast vollständig mit vorhandener Hardware** auskommt: Klon auf eine 512-GB-NVMe, alte Systemplatte als Notfall-Klon, Backup auf externer HDD ([Kapitel 07](07-backup-veeam.md)). Neu gekauft wurden nur zwei USB-Gehäuse.
>
> **Status:** ✅ umgesetzt am 14.11.2026 – C: von 237 GB (3 % frei) auf 475 GB (51 % frei).

---

## 1. Ausgangslage

Der Host ist ein Lenovo ThinkCentre M720q (Tiny-Format), auf dem alle VMs des Labs laufen:

| VM | Rolle |
|---|---|
| DC01 | Windows Server 2022, AD DS, DNS (homelab.local) |
| Windows-11-Client | Domänen-Client, RSAT |
| Zammad | Ubuntu Server, Docker Compose, nginx + interne CA |

**Hardware-Ist-Stand (13.11.2026):**

| Komponente | Ausstattung |
|---|---|
| CPU | Intel Core i3-8100T, 4 Kerne / 4 Threads, 35 W |
| RAM | 16 GB (2× 8 GB DDR4-2666 SO-DIMM), beide Slots belegt |
| Systemplatte | WD PC SN730 NVMe, 256 GB |
| Erweiterung | 1× M.2 (belegt), 2,5"-SATA-Schacht (leer), PCIe x8 (nur mit Riser) |

Erhoben per PowerShell (`Win32_Processor`, `Win32_PhysicalMemory`, `Get-PhysicalDisk`, `Win32_BIOS`) und Sichtprüfung bei geöffnetem Gehäuse.

## 2. Problem

- Die 256-GB-Systemplatte war durch VM-Dateien und **Snapshots** nahezu voll.
- **Auslöser:** Beim Aufbau des AD-Rollenmodells sollte vor der L2-Delegation Platz geschaffen werden. Selbst das **Löschen** eines alten Snapshots schlug fehl: VirtualBox muss dabei die Differenzplatte in die Basisplatte zusammenführen und braucht dafür freien Speicher (`E_OUTOFMEMORY`, „Not enough free storage space“). Ohne freien Platz ließen sich Snapshots also weder anlegen noch entfernen. Die Arbeit wurde daraufhin **bewusst abgebrochen** – ohne Rückfallpunkt keine weiteren Änderungen am Domain Controller.

  ![Snapshot löschen schlägt fehl – nicht genug Speicher für das Zusammenführen](../screenshots/90-virtualbox-snapshot-merge-kein-speicher.png)
- Snapshots sind für das Lab zentral (Rückfallpunkt vor jedem Umbau) – ohne freien Platz entfällt diese Absicherung.
- Ein geplantes Backup-Konzept (Veeam) braucht zusätzlich ein **separates** Sicherungsziel.

## 3. Bestandsaufnahme vorhandener Teile

Bevor gekauft wird, wurde geprüft, was bereits vorhanden ist:

| Teil | Typ | Bewertung |
|---|---|---|
| LITE-ON CA3-8D512, 512 GB | M.2 NVMe | geeignet als neue Systemplatte |
| Seagate ST1000LM024, 1 TB | 2,5" SATA-**HDD**, 5400 U/min | zu langsam für VMs, gut für Backups |
| LITE-ON L8T-256L9G, 256 GB (2014) | M.2 **SATA** | veraltet, nur Ablage |
| WD SN730, 256 GB (nach Klon frei) | M.2 NVMe | bleibt vorerst als startfähiger Notfall-Klon liegen |

**Lernpunkt:** Eine zunächst als „1-TB-SSD“ eingeplante Platte stellte sich anhand der Modellnummer als HDD heraus. Ohne diese Prüfung wären VMs auf einem ungeeigneten Datenträger gelandet.

## 4. Geprüfte Varianten

| # | Variante | Neukosten | Bewertung |
|---|---|---|---|
| A | Zweite NVMe in Slot „SSD2“ | – | **verworfen:** Slot auf dieser Platinenvariante nicht bestückt (nur Lötpads) |
| B | Klon 256 → 512 GB intern, Altplatten extern | nur USB-Gehäuse | **gewählt** |
| C | Neue 1–2-TB-NVMe kaufen | mittel | technisch gut, aber unnötig, solange 512 GB vorhanden sind |
| D | 2,5"-SATA-SSD in den freien Schacht | gering–mittel | sinnvolle **nächste Stufe**, aktuell nicht nötig |
| E | PCIe-Riser + M.2-Adapter | mittel, fummelig | erst relevant, wenn D nicht reicht |
| F | VMs in die Cloud auslagern | laufend | Azure-Testzeitraum abgelaufen; Free-Tiers für AD-Lab ungeeignet |

**Variante A** zeigt, warum Sichtprüfung vor Planung kommt: Das Gehäuse ist für zwei NVMe ausgelegt, die Platine aber nicht bestückt.

## 5. Entscheidung

**Variante B**, nach dem Prinzip *erst vorhandene Ressourcen ausschöpfen, dann gezielt kaufen*.

```mermaid
flowchart LR
    subgraph Intern
        A[LITE-ON 512 GB NVMe<br/>System + laufende VMs]
    end
    subgraph Extern_USB
        C[Seagate 1 TB HDD<br/>Veeam-Backups]
    end
    subgraph Offline
        B[WD SN730 256 GB NVMe<br/>Notfall-Klon vom 14.11.]
    end
    A -- Backup --> C
    A -. Klon .-> B
```

**Begründung:**
- **Wirtschaftlich:** Neukosten nur für zwei USB-Gehäuse; Speicher verdoppelt sich intern.
- **Leistung:** Laufende VMs bleiben auf interner NVMe. Externe USB-Datenträger werden bewusst **nicht** für laufende VMs genutzt (Risiko beschädigter virtueller Platten bei Verbindungsabbruch).
- **Sicherheit:** Backup liegt auf einem **getrennten, abziehbaren** Datenträger – schützt auch gegen Ransomware und Hardwaredefekt des Hosts.
- **Rückfallplan:** Die alte WD-Platte bleibt unverändert als startfähiger Klon liegen. ISOs und Vorlagen liegen nach der Erweiterung auf C:, eine externe Ablage ist vorerst nicht nötig.

## 6. Umsetzung

1. ✅ SMART-Zustand aller Altplatten prüfen (CrystalDiskInfo): alle „Gut“, keine wiederzugewiesenen oder schwebenden Sektoren
2. ✅ BitLocker-Status prüfen (`manage-bde -status`): nicht aktiv, kein Anhalten nötig
3. ✅ Klon WD 256 GB → LITE-ON 512 GB mit Rescuezilla (Boot-Stick via balenaEtcher), Dauer 104,7 Minuten
4. ✅ Platten getauscht, Windows startet von der neuen Platte
5. ✅ C: erweitert: Die Wiederherstellungspartition lag zwischen C: und dem freien Platz und wurde per `reagentc` und `diskpart` ans Plattenende verschoben (siehe unten)
6. WD bleibt als Notfall-Klon offline
7. ✅ 1-TB-HDD als Ziel für Veeam Agent for Microsoft Windows (Free), Backup-Job + **Test-Restore** → eigenes Kapitel [07 – Backup mit Veeam](07-backup-veeam.md)
8. USB Selective Suspend deaktivieren (keine schlafenden Datenträger)


### Nachweise der Vorbereitung

**SMART-Prüfung (CrystalDiskInfo):** Alle Platten „Gut“. Bei der HDD sind die kritischen Werte 05, C4, C5 und C6 bei 0; der CRC-Zähler (C7 = 7) deutet auf frühere Übertragungsfehler über Kabel/Anschluss hin und wird nach den Backup-Läufen beobachtet. Die LITE-ON ist praktisch neu (100 %, 1.084 Betriebsstunden, keine Medienfehler).

![CrystalDiskInfo Seagate 1 TB](../screenshots/92a-crystaldiskinfo-seagate-1tb.png)
![CrystalDiskInfo Seagate: C5/C6 = 0](../screenshots/92b-crystaldiskinfo-seagate-1tb-c5-c6.png)
![CrystalDiskInfo LITE-ON 512 GB](../screenshots/93-crystaldiskinfo-liteon-512gb.png)

**BitLocker:** nicht aktiv, daher kein Anhalten und keine Schlüsselsicherung vor dem Klon nötig.

![manage-bde -status: vollständig entschlüsselt](../screenshots/91-bitlocker-status-nicht-aktiv.png)

**Klon mit Rescuezilla** (Bildschirmfotos, da im Live-System keine Screenshots möglich): alle vier Partitionen übertragen, EFI-Booteintrag aktualisiert, Dauer 104,7 Minuten.

![Rescuezilla: Klon bei 100 %](../screenshots/94-rescuezilla-klon-100-prozent.jpg)
![Rescuezilla: Klon-Zusammenfassung](../screenshots/95-rescuezilla-klon-zusammenfassung.jpg)

### Ergebnis Plattentausch

| | vorher | nachher |
|---|---|---|
| Systemplatte | WD SN730, 256 GB | LITE-ON CA3-8D512, 512 GB |
| C: | 237,45 GB | 475,83 GB |
| frei | 6,37 GB (3 %) | 244,52 GB (51 %) |

**Stolperstein: C: ließ sich nicht erweitern.** Der Klon übernimmt die Partitionen 1:1. Hinter C: lag weiterhin die Wiederherstellungspartition (933 MB) und trennte C: vom freien Platz. Lösung, ohne das Wiederherstellungsabbild zu verlieren:

```
reagentc /disable
diskpart
  select disk 0
  select partition 4
  delete partition override
  select partition 3
  extend
  shrink desired=1024 minimum=1024
  create partition primary id=de94bba4-06d1-4d40-a16a-bfd50179d6ac
  gpt attributes=0x8000000000000001
  format quick fs=ntfs label="Windows RE tools"
reagentc /enable
```

Nachweis: `reagentc /info` meldet „Enabled“, Speicherort `partition4` am Plattenende.

**Vorher** (C: voll, freier Platz hinter der Wiederherstellungspartition):

![Datenträgerverwaltung nach dem Klon](../screenshots/96-datentraeger-nach-klon-c-nicht-erweiterbar.png)
![diskpart: Partition 4 (Wiederherstellung) hinter C:](../screenshots/97-diskpart-partitionen-winre-hinter-c.png)

**Umsetzung und Nachweis:**

![diskpart: C: erweitert, neue Wiederherstellungspartition](../screenshots/98-diskpart-c-erweitert-winre-neu.png)
![reagentc /info: Enabled, partition4](../screenshots/99-reagentc-info-enabled.png)

**Nachher:**

![Datenträgerverwaltung nachher: C: 475,83 GB, 51 % frei](../screenshots/100-datentraeger-nachher-475gb.png)

## 7. Ausblick (bewusst zurückgestellt)

| Maßnahme | Nutzen | Auslöser |
|---|---|---|
| RAM: 1× 16 GB + 1× 8 GB = 24 GB (Sparvariante) | mehr parallele VMs | sobald 16 GB limitieren (absehbar) |
| CPU: i5-9500T / i7-8700T (gebraucht) | 6 statt 4 Kerne | bei CPU-Engpass unter Last |
| 2,5"-SATA-SSD im freien Schacht | weiterer interner VM-Speicher | wenn 512 GB wieder knapp werden |
| LAN-Kabel statt WLAN | stabilere Anbindung | jederzeit, sehr günstig |

RAM wird wegen der aktuellen Preise bewusst als **Sparvariante** geplant: ein 8-GB-Riegel wird durch 1× 16 GB ersetzt (24 GB gesamt). Gemischte Riegelgrößen laufen nur teilweise im Dual-Channel – für ein Lab ein vertretbarer Kompromiss.

## 8. Fazit

Die Erweiterung löst den Speicherengpass mit minimalen Kosten, trennt Betrieb, Ablage und Backup sauber voneinander und hält die nächsten Ausbaustufen offen. Entscheidend war die Reihenfolge **Bestandsaufnahme → Variantenvergleich → Entscheidung** statt direkten Neukaufs.
