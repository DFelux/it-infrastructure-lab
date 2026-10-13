Ziel dieses Projekts ist der praktische Aufbau und Betrieb einer kleinen IT-Infrastruktur im Homelab, um die theoretisch erlernten Inhalte aus dem Google IT Support Professional Certificate praktisch anzuwenden und zu dokumentieren. Das Projekt bildet typische Aufgaben eines Service-Desk-Mitarbeiters ab: Benutzerverwaltung über Active Directory, Support-Fallbearbeitung über ein Ticketsystem sowie grundlegende Windows-Client/Server-Administration.

Das Lab wird schrittweise um Netzwerk- und Security-Komponenten erweitert (virtuelle Firewall, interne CA mit TLS, VLAN-Segmentierung, Netzwerkanalyse). Der aktuelle Stand steht im Abschnitt [Netzwerk- und Security-Erweiterung](#-netzwerk--und-security-erweiterung-in-arbeit).

# Architektur
<img width="1001" height="641" alt="image" src="https://github.com/user-attachments/assets/cbb7d7ce-84e4-461b-b655-7d440ce34069" />


# Umgesetzter Stand
- [x] Windows Server 2022 installiert und konfiguriert
- [x] Active Directory Domain Services (AD DS), Domäne homelab.local
- [x] OU-Struktur (IT, Support, Vertrieb) angelegt
- [x] Erster Benutzer angelegt
- [x] Windows-11-Client installiert, Netzwerk konfiguriert
- [x] Server und Client können sich gegenseitig erreichen (Ping erfolgreich)
- [x] Windows-Client in Domäne aufgenommen
- [x] Ticketsystem eingerichtet
- [x] Ticketsystem-VM (Zammad) neu aufgebaut (v2), Weboberfläche erreichbar, Rollen Support L1–L3 angelegt
- [ ] Zammad konfigurieren: Gruppen, Testbenutzer, Eskalationsablauf L1–L3
- [ ] Virtuelle Firewall (OPNsense) für das interne Netz (geplant)
- [ ] Interne CA und TLS für Zammad über Reverse Proxy (geplant)
- [ ] VLAN-Segmentierung (geplant)


# Was ich dabei gelernt habe

Beim Einrichten des Windows Servers bin ich auf ein Netzwerkproblem gestoßen: 
der zweite Netzwerkadapter war in VirtualBox nicht aktiviert, wodurch die 
Domänenkommunikation zunächst nicht funktionierte. Dies hat mir gezeigt, wie 
wichtig es ist, Netzwerkkonfigurationen systematisch zu prüfen, bevor man 
nach komplexeren Ursachen sucht.



## Troubleshooting: Schwarzer Bildschirm bei Windows-11-Client-VM

Beim ersten Start der Windows-11-Client-VM blieb der Bildschirm in VirtualBox 
dauerhaft schwarz, obwohl die CPU-Auslastung im Hintergrund leicht schwankte – 
ein Hinweis darauf, dass die VM lief, aber nichts angezeigt wurde.

**Vorgehen:**
1. Grafikcontroller auf VMSVGA umgestellt, 3D-Beschleunigung deaktiviert – 
   keine Besserung
2. VBox.log gezielt nach Fehlern durchsucht (Suche nach "Error", "NEM", "EFI")
3. Fündig geworden: `HM: HMR3Init: Attempting fall back to NEM: VT-x is not available` 
   – VirtualBox konnte nicht direkt auf die Hardware-Virtualisierung (VT-x) 
   zugreifen und wich auf einen langsameren Software-Fallback aus
4. Windows Core Isolation / Memory Integrity deaktiviert – Problem blieb bestehen
5. `bcdedit /set hypervisorlaunchtype off` gesetzt – Problem blieb weiterhin bestehen
6. BIOS-Virtualisierung geprüft (Task-Manager → Leistung → CPU) – VT-x war 
   bereits aktiviert, also keine BIOS-Ursache
7. Über `msinfo32` festgestellt, dass "Virtualisierungsbasierte Sicherheit" 
   trotz aller Änderungen weiterhin lief, zusätzlich war "App Control for 
   Business" als Richtlinie erzwungen – ein Hinweis auf verbliebene 
   Enterprise-Sicherheitsrichtlinien (Device Guard/Credential Guard) aus der 
   vorherigen Nutzung des generalüberholten Business-Geräts

**Ursache:** Das gebraucht gekaufte Lenovo ThinkCentre M720q lief zuvor in 
einer Unternehmensumgebung mit aktivierten Device-Guard-/Credential-Guard-
Richtlinien. Diese Richtlinien blieben auch nach dem Refurbishing über die 
lokale Gruppenrichtlinie aktiv und aktivierten Windows' eigene 
Virtualisierungsschicht (Hyper-V/VBS) bei jedem Systemstart neu – dadurch 
stand VirtualBox kein direkter VT-x-Zugriff mehr zur Verfügung.

**Lösung:** Über die lokale Gruppenrichtlinie (`gpedit.msc` → 
Computerkonfiguration → Administrative Vorlagen → System → Device Guard) 
"Auf Virtualisierung basierende Sicherheit aktivieren" explizit auf 
**Deaktiviert** gesetzt (nicht nur "Nicht konfiguriert"), anschließend 
`gpupdate /force` und Neustart des Hosts. Danach lief die VM ohne 
Einschränkungen.

**Erkenntnis:** Bei gebrauchter Business-Hardware können Sicherheits-
richtlinien aus der vorherigen Nutzung tief im System verankert bleiben und 
sich nicht durch einfache Einstellungsänderungen beheben lassen. Die 
eigentliche Ursache ließ sich erst durch systematisches Auswerten der 
VirtualBox-Logs und der Windows-Systeminformationen statt durch reines 
Ausprobieren finden.

## Troubleshooting: IP-Adresskonflikt zwischen Host und Server-VM

Nach der Netzwerkkonfiguration von Server und Client meldete Windows Server 
bei seiner eigenen Host-only-IP (`192.168.56.10`) dauerhaft den Status 
„Dupliziert" und wich auf eine automatische Behelfsadresse (169.254.x.x) aus. 
Ping-Versuche zwischen Server und Client schlugen dadurch in beide 
Richtungen fehl.

**Vorgehen:**
1. Feste IP-Konfiguration auf Server und Client einzeln über `ipconfig /all` 
   geprüft – beide korrekt eingetragen, kein Tippfehler
2. Mit `arp -a` vom Client aus geprüft, welches Gerät tatsächlich auf 
   `192.168.56.10` antwortet
3. Die dabei ermittelte MAC-Adresse mit den bekannten Adaptern abgeglichen – 
   sie gehörte weder zum Server noch zum Client, sondern zum 
   **VirtualBox Host-Only-Adapter auf dem Host-PC selbst**
4. Festgestellt: Bei der vorherigen Fehlersuche wurde versehentlich auf dem 
   Host-Adapter (statt innerhalb der Server-VM) eine manuelle IP im selben 
   Adressbereich gesetzt – leicht passiert, da mehrere verschachtelte 
   Remotedesktop-/VM-Fenster gleichzeitig offen waren

**Ursache:** Zwei Geräte im selben internen Netzwerk (Host-PC und Server-VM) 
beanspruchten dieselbe IP-Adresse, weil die Netzwerkeinstellungen versehentlich 
auf der falschen Ebene (Host statt Gast-VM) geändert wurden.

**Lösung:** Die manuelle IP-Konfiguration auf dem Host-only-Adapter des 
Host-PCs zurück auf automatischen Bezug gestellt, sodass ausschließlich 
Server (`192.168.56.10`) und Client (`192.168.56.20`) feste Adressen im 
internen Netz halten. Nach einem Neustart des Adapters funktionierte die 
Kommunikation zwischen Server und Client fehlerfrei.

**Erkenntnis:** Bei verschachtelten Umgebungen (Host, Remotedesktop, mehrere 
VM-Fenster gleichzeitig) ist es entscheidend, vor jeder Netzwerkänderung zu 
prüfen, auf welcher Ebene man sich gerade befindet. Ein `arp -a` vom 
unbeteiligten dritten Gerät aus (hier: Client) war der entscheidende 
Schritt, um die Ursache eindeutig zu lokalisieren, statt nur auf dem 
betroffenen Server selbst zu suchen.

## Meilenstein: Client erfolgreich in Domäne aufgenommen

Nach Behebung des IP-Konflikts konnte der Windows-11-Client erfolgreich der 
Domäne `homelab.local` beitreten. Nach einem Neustart erfolgt der Login nun 
über das Domänenkonto (`HOMELAB\Administrator`), der Client erscheint 
korrekt im Container „Computers" der Active-Directory-Struktur auf dem Server.

## Screenshots

### VirtualBox – Snapshot-Verlauf
![Snapshot-Baum](screenshots/01-vm-snapshot-baum.png)

Jeder wichtige Meilenstein wurde als Snapshot gesichert – von der 
Frischinstallation bis zum Domänenbeitritt des Clients.

### Server- und Client-VM im Überblick
![VM-Details Server](screenshots/02-vm-details-server.png)
![VM-Details Client](screenshots/03-vm-details-client.png)

### VM-Erstellung: Ressourcenzuteilung
![VM-Ressourcen](screenshots/04-vm-erstellung-ressourcen.png)

### Server-Manager: Lokaler Server
![Server-Manager](screenshots/05-server-manager-lokaler-server.png)

### Netzwerkadapter-Konfiguration (Host-only)
![Netzwerkadapter](screenshots/06-netzwerkadapter-einstellungen.png)
![IPv4-Eigenschaften Ethernet 2](screenshots/07-ipv4-eigenschaften-ethernet2.png)

Feste IP-Adressen im internen Netz vergeben: Server `192.168.56.10`, 
Client `192.168.56.20`.

### Troubleshooting: IP-Adresskonflikt

**Fehlerbild:** Der zweite Netzwerkadapter (NAT) hatte versehentlich eine 
leere, ungültige manuelle IP-Konfiguration.
![Ethernet 1 leer](screenshots/08-ipv4-eigenschaften-ethernet1-leer.png)

**Symptom:** Der Server zeigte bei seiner eigenen IP den Status „Dupliziert“, 
Ping zwischen Server und Client schlug fehl.
![Ping-Timeout](screenshots/15-ping-timeout-fehlerfall.png)
![ipconfig Duplikat](screenshots/11-ipconfig-server-duplikat.png)

**Diagnose:** Mit `arp -a` vom Client aus die tatsächlich antwortende 
MAC-Adresse ermittelt und mit den bekannten Adaptern abgeglichen.

![arp -a Client](screenshots/14-arp-a-client-ursache-gefunden.png)

![arp -a Server](screenshots/13-arp-a-server-seite.png)

**Lösung:** Nach Korrektur der Host-Adapter-Konfiguration funktionierte die 
Kommunikation fehlerfrei.
![ipconfig korrekt Client](screenshots/12-ipconfig-client-korrekt.png)

![ipconfig NAT repariert](screenshots/10-ipconfig-nat-adapter-repariert.png)

![ipconfig finaler Check](screenshots/16-ipconfig-final-check.png)

### Server-Identität nach Umbenennung (DC01)
![ipconfig DC01](screenshots/09-ipconfig-server-dc01.png)

## Meilenstein: Ticketsystem-VM (Ubuntu Server + Docker)

Als dritte Komponente wurde eine Ubuntu Server 26.04 LTS VM aufgesetzt, 
auf der Zammad containerisiert über Docker Compose betrieben wird.

### Troubleshooting: Netplan-Konfiguration schlägt trotz korrektem Inhalt fehl

**Fehlerbild:** `netplan apply` meldete wiederholt 
`expected mapping (check indentation)`, obwohl die eingefügte YAML-Datei 
augenscheinlich korrekt aussah.

**Ursache:** Die VirtualBox-Textkonsole (ohne grafische Oberfläche) 
entfernt beim Einfügen mehrzeiligen Texts die führenden Leerzeichen jeder 
Zeile – dadurch ging die für YAML zwingend nötige Einrückung verloren.

**Lösung:** Die Konfigurationsdatei stattdessen über einen kontrollierten 
`printf`-Befehl in einer einzigen Zeile geschrieben, sodass Einrückung 
und Zeilenumbrüche nicht durch das Einfügeverhalten der Konsole verändert 
werden konnten.

### Troubleshooting: Host verliert Verbindung zu allen Lab-VMs

**Fehlerbild:** Nach einem vorherigen Fix (Host-Adapter auf automatischen 
IP-Bezug zurückgesetzt) konnte der Host-PC plötzlich weder Server, Client 
noch die neue Ticketsystem-VM erreichen.

**Diagnose:** `ipconfig /all` zeigte beim Host-only-Adapter eine 
APIPA-Adresse (169.254.x.x) statt einer gültigen IP – der Adapter 
versuchte per DHCP eine Adresse zu beziehen, bekam aber keine Antwort, 
da der DHCP-Server im internen Netz bewusst deaktiviert ist.

**Lösung:** Dem Host-Adapter eine eigene, feste IP (192.168.56.1) 
zugewiesen, die nicht mit Server (.10), Client (.20) oder Ticketsystem 
(.30) kollidiert.

**Erkenntnis:** Bei einem selbst verwalteten internen Netzwerk ohne 
DHCP-Server benötigt jedes teilnehmende Gerät – auch der Host selbst – 
eine bewusst vergebene, feste IP-Adresse.

## Neuaufbau der Ticketsystem-VM (Zammad v2)

Die Ticketsystem-VM (`ZammadTicketv2`) wurde neu aufgesetzt, diesmal 
bewusst selbstständig anhand der offiziellen Online-Dokumentation. Die VM 
(Ubuntu Server 26.04) hat zwei Netzwerkadapter: `enp0s3` für den 
Internetzugang (Updates, Docker-Images) und `enp0s8` im internen Host-only-Netz 
mit der festen Adresse `192.168.56.30`. Der Zugriff vom Notebook erfolgt über 
NAT mit Portweiterleitung (siehe unten); zwischenzeitlich wurde eine 
Netzwerkbrücke verwendet.

Ressourcen der VM: 8 GB RAM, 2 Prozessoren, virtuelle Platte mit 25 GB. 
Der Überblick zeigt alle VMs des Labs.
![VirtualBox-Überblick](screenshots/37-virtualbox-ueberblick-alle-vms.png)

### Troubleshooting: Fehlermeldung bei der Netzwerkkonfiguration im Ubuntu-Installer

**Fehlerbild:** Der Installer lehnte die statische Adresse mit 
`'192.168.56.30' is not contained in '255.255.255.0/24'` ab.
![Installer Fehlermeldung](screenshots/17-zammad-installer-subnet-fehler.png)

**Ursache:** Zwei Fehler gleichzeitig:
1. Im Feld *Subnet* war die Subnetzmaske (`255.255.255.0/24`) eingetragen, 
   der Installer erwartet dort die **Netzadresse in CIDR-Schreibweise** 
   (`192.168.56.0/24`).
2. Die Interfaces waren vertauscht: `enp0s3` ist der NAT-Adapter 
   (DHCP, `10.0.2.15/24`), das Host-only-Netz gehört zu `enp0s8`. Eine 
   statische Host-only-Adresse auf `enp0s3` hätte den Internetzugang der VM 
   gekappt.

**Lösung:** `enp0s3` unverändert per DHCP belassen, `enp0s8` auf *Manual* 
gestellt mit Subnet `192.168.56.0/24` und Address `192.168.56.30`, 
Gateway und Name servers leer (im Host-only-Netz gibt es keines).
![Netzwerk final im Installer](screenshots/18-zammad-installer-netzwerk-final.png)

**Erkenntnis:** Die Zuordnung der Interfaces lässt sich über die 
MAC-Adressen im Installer und die Adapter-Reihenfolge in den 
VirtualBox-Netzwerkeinstellungen (Adapter 1 = NAT, Adapter 2 = Host-only) 
verlässlich prüfen. Nach der Installation bestätigte `ip a` die Konfiguration.
![ip a und SSH-Status](screenshots/19-zammad-ip-a-ssh-status.png)

### Troubleshooting: VM per SSH vom Notebook nicht erreichbar, obwohl alles korrekt konfiguriert ist

**Symptom:** Von der Windows-Server-VM war die Zammad-VM erreichbar, vom 
Notebook (Zugriff auf das Lab) dagegen weder per Ping noch per 
`Test-NetConnection 192.168.56.30 -Port 22` (Timeout). Im Testergebnis fiel 
auf: `InterfaceAlias: WLAN`, `SourceAddress: 192.168.178.27` – die Pakete 
verließen das Notebook über das WLAN statt über den Host-only-Adapter.
![Test-NetConnection über WLAN](screenshots/20-test-netconnection-wlan-fehlschlag.png)

**Vorgehen:**
1. `ipconfig` am Notebook: Der Host-only-Adapter hat `192.168.56.1/24`, 
   liegt also im selben Netz wie die VM – die Konfiguration wirkte stimmig
   ![ipconfig Host-only-Adapter am Notebook](screenshots/21-ipconfig-notebook-host-only.png)
2. `Find-NetRoute -RemoteIPAddress 192.168.56.30`: Windows routet über die 
   Standardroute (WLAN, Fritzbox) statt über den Host-only-Adapter
3. `Get-NetRoute` und `Get-NetAdapter`: Die Route `192.168.56.0/24` existiert, 
   der Adapter ist `Up` und `Connected` – der Adapter selbst ist also intakt
4. `ping -S 192.168.56.1 192.168.56.30` (Quelladresse erzwungen) lieferte 
   „Zielhost nicht erreichbar“ vom Absender `192.168.56.1` selbst – die 
   ARP-Auflösung bleibt unbeantwortet, das Problem liegt auf Layer 2 und 
   weder bei Firewall noch bei Routing
5. Gegenprobe in der VM: `ping 192.168.56.1` funktioniert (`ttl=128`), 
   `ip neigh` zeigt für `192.168.56.1` die MAC `0a:00:27:00:00:14`
   ![ip neigh in der VM](screenshots/22-vm-ip-neigh-mac-abweichung.png)
6. `Get-NetAdapter` am Notebook: Der Host-only-Adapter hat die MAC 
   `0A-00-27-00-00-0D` – **nicht** dieselbe MAC, die die VM sieht

**Ursache:** Die VM läuft nicht auf dem Notebook, sondern auf dem Mini-PC 
(Lenovo ThinkCentre M720q). Ein Host-only-Netz existiert nur auf dem Rechner, 
auf dem VirtualBox läuft. Das Notebook besitzt einen eigenen, völlig 
getrennten Host-only-Adapter, der zufällig dieselbe Standardadresse 
`192.168.56.1` trägt. Dass die Windows-Server-VM die Zammad-VM erreicht, 
liegt daran, dass beide im Host-only-Netz des Mini-PCs hängen.

**Lösung (Zwischenlösung):** In der VM-Konfiguration den Adapter 1 von NAT auf 
**Netzwerkbrücke** (Bridged) umgestellt. Die VM erhielt eine Adresse aus 
dem Heimnetz (`192.168.178.31`) und war vom Notebook per SSH erreichbar. 
Adapter 2 (`enp0s8`, `192.168.56.30`) blieb unverändert für die interne 
Kommunikation mit Server und Client. Die Brücke wurde später wegen sehr 
langsamer Downloads wieder durch NAT mit Portweiterleitung ersetzt (siehe 
weiter unten).
![ip a mit Bridged-Adapter](screenshots/23-zammad-ip-a-bridged.png)

**Erkenntnis:** Host-only-Netze sind auf den jeweiligen Host-Rechner 
begrenzt; derselbe Standardadressbereich kann auf mehreren Rechnern 
gleichzeitig existieren und täuscht dann Erreichbarkeit vor. Entscheidend 
war der **Vergleich der MAC-Adressen** zwischen Gast (`ip neigh`) und Host 
(`Get-NetAdapter`). Ein erzwungener Ping mit `-S` schloss Firewall und 
Routing sauber aus, bevor die Ursache gefunden wurde.

### Troubleshooting: Zammad-Weboberfläche nicht erreichbar („Verbindung abgelehnt“)

**Symptom:** In der Fritzbox erscheint die VM, die Schaltfläche 
„Konfiguration“ liefert „Fehler 404 – Seite nicht gefunden“. Im Browser meldet 
`192.168.178.31` „Verbindung abgelehnt“.

**Diagnose:**
1. „Verbindung abgelehnt“ (statt Timeout) bedeutet: Die VM ist erreichbar, 
   aber auf dem angefragten Port lauscht kein Dienst – eine Firewall wäre es nicht
2. `ss -tlnp`: Es lauschen nur SSH (Port 22) und der lokale DNS-Stub, 
   nichts auf Port 80 oder 8080
3. `docker compose ps` meldete `docker: unknown command: docker compose`
   ![ss -tlnp und fehlendes Compose-Plugin](screenshots/24-zammad-ss-tlnp-compose-fehlt.png)

**Ursache:** Das Docker-Compose-Plugin fehlt, der Zammad-Stack wurde 
deshalb nie gestartet. Die Schaltfläche der Fritzbox ruft Port 80 auf, auf dem 
die VM nichts anbietet; die 404-Seite stammt vermutlich von der Fritzbox selbst.

**Erkenntnis:** Der Unterschied zwischen „Verbindung abgelehnt“ (Dienst 
lauscht nicht) und Timeout (Paket kommt nicht an / wird gefiltert) grenzt die 
Fehlersuche sofort ein, `ss -tlnp` belegt es.

### Troubleshooting: Docker-Repository lässt sich nicht einbinden

**Fehlerbild:** Bei der Installation von `docker-ce`, `docker-compose-plugin` 
und Co. meldete `apt`: „Für Paket ›docker-ce‹ existiert kein 
Installationskandidat“. Zuvor hatte der Befehl zum Anlegen des Schlüssels 
`gpg: no valid OpenPGP data found` ausgegeben.
![Fehlermeldungen bei der Docker-Installation](screenshots/25-docker-repo-gpg-fehler.png)

**Diagnose und Behebung:**
1. Der Schlüssel wurde mit `curl` von `https://docker.com` geladen – das 
   liefert die Webseite (HTML), keinen GPG-Schlüssel. Richtig ist 
   `https://download.docker.com/linux/ubuntu/gpg`.
2. `apt update` meldete für `https://www.docker.com resolute` „404 Not Found“ 
   und „enthält keine Release-Datei“. In `/etc/apt/sources.list.d/docker.list` 
   stand die falsche URL `https://docker.com`. Gefunden mit 
   `grep -rn "docker" /etc/apt/sources.list /etc/apt/sources.list.d/`, die 
   Datei wurde gelöscht.
   ![Falscher Repository-Eintrag gefunden und entfernt](screenshots/26-docker-falsches-repo-entfernt.png)
3. Neuer Eintrag `/etc/apt/sources.list.d/docker.sources` mit 
   `URIs: https://download.docker.com/linux/ubuntu` und `Suites: resolute`. 
   `apt update` lud nun die Paketliste von `download.docker.com` – das 
   Docker-Repository unterstützt Ubuntu 26.04 also – meldete aber 
   `NO_PUBKEY`, weil die Schlüsseldatei `/etc/apt/keyrings/docker.asc` nie 
   angelegt worden war (der `curl`-Befehl war beim fehlerhaften Einfügen 
   untergegangen, siehe nächster Abschnitt).
   ![NO_PUBKEY-Fehler](screenshots/27-docker-no-pubkey-fehler.png)
4. Schlüssel von der richtigen URL nachgeladen, `apt update` lief ohne 
   Fehler. Danach `docker-ce`, `docker-ce-cli`, `containerd.io`, 
   `docker-buildx-plugin` und `docker-compose-plugin` installiert. Dabei 
   wurden die zuvor installierten Ubuntu-Pakete `docker.io`, `containerd` 
   und `runc` ersetzt.
   ![Installation ersetzt docker.io](screenshots/28-docker-installation-ersetzt-docker-io.png)
5. Test mit `sudo docker run hello-world` erfolgreich.
   ![hello-world](screenshots/29-docker-hello-world.png)

**Ursache:** Mehrere Fehler zusammen: falsche Schlüssel-URL und falsche 
Repository-URL (jeweils ohne `download.`), eine nie angelegte Schlüsseldatei 
und ein zuvor installiertes `docker.io` aus den Ubuntu-Quellen, das kein 
Compose-Plugin mitbrachte.

**Erkenntnis:** `NO_PUBKEY` zeigt, dass die Quelle stimmt und nur der 
Schlüssel fehlt; „404“ und „keine Release-Datei“ weisen dagegen auf eine 
falsche Repository-URL hin. Das offizielle Docker-Repository bietet Pakete 
für Ubuntu 26.04 (`resolute`) an.

### Troubleshooting: Befehle lassen sich nicht sauber einfügen

**Fehlerbild:** Beim Einfügen mehrzeiliger Befehlsblöcke in die SSH-Sitzung 
tauchten Steuerzeichen im Text auf: `^[[200~` und `^[[201~` vor bzw. hinter 
dem Block sowie `^M` am Zeilenende. Folgen: `sudo: command not found` (aus 
`sudo` wurde `^[[201~sudo`), und ein Heredoc, das nicht endete, weil die 
Shell `EOF^M` nicht als Abschluss erkannte.

**Ursache:** Das Terminal sendet beim Einfügen die Markierungen des 
„Bracketed Paste“-Modus, die als Text ankamen, und die Zeilenenden aus der 
Windows-Zwischenablage wurden als `^M` übergeben.

**Lösung:**
- Für die Sitzung `bind 'set enable-bracketed-paste off'`
- Befehle einzeln statt als Block einfügen, eine Passwortabfrage vorher mit 
  `sudo -v` erledigen
- Konfigurationsdateien statt per Heredoc mit einem einzeiligen 
  `printf … | sudo tee …` schreiben (wie schon bei der Netplan-Datei)

**Erkenntnis:** Auch eine SSH-Sitzung löst das Einfügeproblem der 
VM-Konsole nicht automatisch. Entscheidend ist, eingefügte Befehle vor dem 
Absenden auf Steuerzeichen zu prüfen und das Ergebnis jedes Schritts zu 
kontrollieren (`cat`, `head`).

### Troubleshooting: „permission denied“ beim Zugriff auf den Docker-Socket

**Fehlerbild:** `docker compose up` meldete 
`permission denied while trying to connect to the docker API at unix:///var/run/docker.sock`. 
Die Warnung zu `postgresql-initdb.mode` („not supported outside Swarm mode“) 
ist dagegen harmlos und wird ignoriert.

**Ursache:** Der Benutzer war nicht Mitglied der Gruppe `docker`.

**Lösung:** `sudo usermod -aG docker $USER`, danach ab- und wieder anmelden. 
Anschließend läuft `docker compose` ohne `sudo`.

**Hinweis:** Die Mitgliedschaft in der Gruppe `docker` entspricht praktisch 
Root-Rechten auf dem Host. Im Lab ist das ein bewusster Kompromiss.

### Troubleshooting: Sehr langsamer Image-Download mit Netzwerkbrücke über WLAN

**Symptom:** Der Download der Zammad-Images kam mit der Netzwerkbrücke kaum 
voran. Beim ersten Aufbau der VM (mit NAT) hatte der gesamte Download 
ungefähr 10 Minuten gedauert.
![Image-Pull mit Netzwerkbrücke](screenshots/30-zammad-compose-pull-bridged-wlan.png)

**Diagnose:** Der Mini-PC hängt per WLAN am Router (ein LAN-Kabel ist nicht 
möglich). Die empfangenen Bytes über zehn Sekunden gemessen: 
`cat /sys/class/net/enp0s3/statistics/rx_bytes; sleep 10; cat /sys/class/net/enp0s3/statistics/rx_bytes` 
ergab 4.125.918 Bytes, also rund 410 kB/s. Der Unterschied zum ersten Aufbau 
war der Adaptertyp: damals NAT, jetzt Netzwerkbrücke über WLAN.
![Messung der Downloadrate](screenshots/31-zammad-rx-bytes-messung.png)

**Lösung:** Adapter 1 zurück auf **NAT** gestellt und zwei 
Portweiterleitungen angelegt: Host-Port `2222` auf Gast-Port `22` (SSH) und 
Host-Port `8080` auf Gast-Port `8080` (Zammad). `enp0s8` blieb unverändert.
![Portweiterleitung](screenshots/32-zammad-nat-portweiterleitung.png)
![ip a mit NAT](screenshots/33-zammad-ip-a-nat.png)

Der Zugriff vom Notebook erfolgt nun über die IP des Mini-PCs: 
`ssh -p 2222 <benutzer>@<IP-des-Mini-PCs>` und 
`http://<IP-des-Mini-PCs>:8080`.

**Erkenntnis:** Bei NAT liegt die VM in einem eigenen privaten Netz (`10.0.2.x`) 
hinter dem Host. Vom Notebook aus spricht man deshalb immer den Mini-PC an, 
VirtualBox reicht die Verbindung per Portweiterleitung an die VM weiter. 
Dass die Netzwerkbrücke über WLAN die Ursache war, ergibt sich aus dem 
Vorher-Nachher-Vergleich und nicht aus einer isolierten Messung. Zu beachten: 
Ohne Host-IP in den Regeln sind die Ports von allen Geräten im Heimnetz 
erreichbar.

## Meilenstein: Zammad läuft

Der Stack wird im Verzeichnis `zammad-docker-compose` mit 
`docker compose up -d` gestartet. Im Home-Verzeichnis meldet `docker compose` 
dagegen „no configuration file provided: not found“, denn die Befehle müssen 
dort ausgeführt werden, wo die `docker-compose.yml` liegt. Beim Start 
wurden Netzwerk und Volumes angelegt, Redis, PostgreSQL und Memcached meldeten 
sich als `Healthy`, der Rails-Server wartete auf den Init-Container.
![docker compose up -d](screenshots/34-zammad-compose-up.png)

Anschließend war die Weboberfläche unter `http://<IP-des-Mini-PCs>:8080` 
erreichbar, und der Einrichtungsassistent konnte durchlaufen werden.
![Zammad Dashboard](screenshots/35-zammad-dashboard.png)

Zusätzlich zu den Standardrollen (Admin, Agent, Customer) wurden die Rollen 
**Support L1, L2 und L3** angelegt, um eine Eskalationsstruktur wie in einem 
Service Desk abzubilden.
![Rollen in Zammad](screenshots/36-zammad-rollen.png)

<!-- Berechtigungen und Gruppenzugriff der Rollen nach der Konfiguration ergänzen -->

### Troubleshooting: Weniger Speicherplatz als die virtuelle Platte hat

**Symptom:** Das System zeigte nur rund 11 GB (`Usage of /: … of 11.21GB`), 
obwohl die virtuelle Platte 25 GB groß ist.

**Ursache:** Der Ubuntu-Installer legt das logische Volume standardmäßig nur 
auf einen Teil der Platte, der Rest bleibt in der Volume Group ungenutzt.

**Lösung:** `sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv`. Die 
Option `-r` vergrößert das Dateisystem gleich mit, im laufenden Betrieb und 
ohne Neustart. Das Volume wuchs von 11,5 GiB auf 23 GiB, `df -h /` zeigt 
anschließend 23 GB Gesamtgröße und 14 GB frei.
![lvextend und df](screenshots/38-zammad-lvextend-speicher-erweitert.png)

**Erkenntnis:** Der Installer-Standard nutzt nicht den gesamten Plattenplatz. 
Gerade bei Anwendungen, die wachsen (Zammad mit Datenbank und Suchindex), 
lohnt sich die Kontrolle mit `df -h` und `vgs` nach der Installation.

## 🔐 Netzwerk- und Security-Erweiterung (in Arbeit)

Ausbau des Labs um typische Komponenten einer abgesicherten 
Unternehmensumgebung. Status: **geplant**, einzelne Bausteine werden nach 
Abschluss mit Konfiguration, Screenshots und Troubleshooting dokumentiert. 
Bisher läuft der Zugriff auf Zammad ungeschützt über HTTP und ohne zentrale 
Firewall, genau das soll dieser Ausbau ändern.

| Baustein | Ziel | Status |
|---|---|---|
| Virtuelle Firewall (OPNsense/pfSense) | Zentrale Absicherung des internen Netzes, Regelwerk, NAT/PAT, Portweiterleitung | ⬜ geplant |
| Interne CA und TLS-Zertifikate | X.509-Zertifikate, TLS für Zammad über Reverse Proxy (nginx/Caddy) | ⬜ geplant |
| VLAN-Segmentierung | Trennung von Server-, Client- und Management-Netz | ⬜ geplant |
| Netzwerkanalyse | Paketmitschnitte mit tcpdump/Wireshark, DNS-Prüfung mit dig | ⬜ geplant |
| Konfig-Backup und Restore | Sicherung und Wiederherstellung der Firewall-Konfiguration | ⬜ geplant |

### Lernziele
- Firewall-Regeln planen, testen und anhand der Logs nachvollziehen
- Aufbau und Betrieb einer einfachen CA-Struktur, Ausstellen und Einbinden von Zertifikaten
- Fehlersuche auf Netzwerk- und Zertifikatsebene

### Dokumentation je Baustein
Ziel · Aufbau (Netzwerkdiagramm) · Konfiguration · Tests · Troubleshooting · Screenshots

## Nächste Schritte

- Gruppen anlegen und den Rollen L1 bis L3 Gruppenzugriff zuweisen
- Testbenutzer (Agent und Kunde) und Testtickets anlegen, Eskalation von L1 
  an L2 durchspielen
- Zugriff absichern: Zammad läuft derzeit über HTTP („Nicht sicher“ im 
  Browser), ein vorgeschalteter Reverse Proxy mit TLS wäre der nächste Ausbau 
  (siehe [Netzwerk- und Security-Erweiterung](#-netzwerk--und-security-erweiterung-in-arbeit))
- Virtuelle Firewall aufsetzen und das interne Netz dahinter betreiben
- SSH-Zugang härten (eigener Benutzername statt Standardnamen, 
  Schlüsselauthentifizierung)
