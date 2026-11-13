# Troubleshooting-Index

[← Zur Übersicht](README.md)

Alle 30 dokumentierten Fehlerfälle mit Ursache, Lösung und Nachweis, nach Teilprojekt sortiert. Jeder Link führt direkt zum Fall.


## Active Directory, Client und Grundaufbau

- [Schwarzer Bildschirm bei Windows-11-Client-VM](docs/01-active-directory.md#troubleshooting-schwarzer-bildschirm-bei-windows-11-client-vm)
- [IP-Adresskonflikt zwischen Host und Server-VM](docs/01-active-directory.md#troubleshooting-ip-adresskonflikt-zwischen-host-und-server-vm)
- [„Zugriff verweigert“ beim Verschieben von OUs](docs/01-active-directory.md#troubleshooting-zugriff-verweigert-beim-verschieben-von-ous)
- [Server-Admin-Konto hatte über vordefinierte Gruppe DC-Vollzugriff](docs/01-active-directory.md#2-befund-server-admin-konto-hatte-über-eine-unauffällige-gruppe-dc-vollzugriff)
- [Keine Kontosperrung in der Domäne (Sperrschwelle 0)](docs/01-active-directory.md#2-befund-keine-kontosperrung-in-der-domäne)
- [RSAT-Installation meldet Erfolg, installiert aber nichts](docs/01-active-directory.md#troubleshooting-rsat-installation-meldet-erfolg-installiert-aber-nichts)
- [„RPC-Server nicht verfügbar“ durch NAT-Adresse des DC im DNS](docs/01-active-directory.md#troubleshooting-rpc-server-nicht-verfügbar-durch-nat-adresse-des-dc-im-dns)

## Ticketsystem Zammad (Ubuntu + Docker)

- [Netplan-Konfiguration schlägt trotz korrektem Inhalt fehl](docs/02-zammad.md#troubleshooting-netplan-konfiguration-schlägt-trotz-korrektem-inhalt-fehl)
- [Host verliert Verbindung zu allen Lab-VMs](docs/02-zammad.md#troubleshooting-host-verliert-verbindung-zu-allen-lab-vms)
- [Fehlermeldung bei der Netzwerkkonfiguration im Ubuntu-Installer](docs/02-zammad.md#troubleshooting-fehlermeldung-bei-der-netzwerkkonfiguration-im-ubuntu-installer)
- [VM per SSH vom Notebook nicht erreichbar, obwohl alles korrekt konfiguriert ist](docs/02-zammad.md#troubleshooting-vm-per-ssh-vom-notebook-nicht-erreichbar-obwohl-alles-korrekt-konfiguriert-ist)
- [Zammad-Weboberfläche nicht erreichbar („Verbindung abgelehnt“)](docs/02-zammad.md#troubleshooting-zammad-weboberfläche-nicht-erreichbar-verbindung-abgelehnt)
- [Docker-Repository lässt sich nicht einbinden](docs/02-zammad.md#troubleshooting-docker-repository-lässt-sich-nicht-einbinden)
- [Befehle lassen sich nicht sauber einfügen](docs/02-zammad.md#troubleshooting-befehle-lassen-sich-nicht-sauber-einfügen)
- [„permission denied“ beim Zugriff auf den Docker-Socket](docs/02-zammad.md#troubleshooting-permission-denied-beim-zugriff-auf-den-docker-socket)
- [Sehr langsamer Image-Download mit Netzwerkbrücke über WLAN](docs/02-zammad.md#troubleshooting-sehr-langsamer-image-download-mit-netzwerkbrücke-über-wlan)
- [Weniger Speicherplatz als die virtuelle Platte hat](docs/02-zammad.md#troubleshooting-weniger-speicherplatz-als-die-virtuelle-platte-hat)

## Interne CA und TLS für Zammad

- [Host-Laufwerk voll (`VERR_DISK_FULL`)](docs/03-pki-tls.md#troubleshooting-host-laufwerk-voll-verr_disk_full)
- [Root-Key hat 0 Bytes (`Verify failure`)](docs/03-pki-tls.md#troubleshooting-root-key-hat-0-bytes-verify-failure)
- [Eingefügte Befehle werden verfälscht](docs/03-pki-tls.md#troubleshooting-eingefügte-befehle-werden-verfälscht)
- [`install: invalid group: 'ssl-cert'`](docs/03-pki-tls.md#troubleshooting-install-invalid-group-ssl-cert)
- [nginx direkt gestartet (`Permission denied` beim Key)](docs/03-pki-tls.md#troubleshooting-nginx-direkt-gestartet-permission-denied-beim-key)
- [SSH-Warnung `REMOTE HOST IDENTIFICATION HAS CHANGED`](docs/03-pki-tls.md#troubleshooting-ssh-warnung-remote-host-identification-has-changed)
- [SSH-Test aus dem Heimnetz läuft in den Timeout](docs/03-pki-tls.md#troubleshooting-ssh-test-aus-dem-heimnetz-läuft-in-den-timeout)
- [PowerShell erkennt den Parameter nicht](docs/03-pki-tls.md#troubleshooting-powershell-erkennt-den-parameter-nicht)
- [`ERR_CERT_AUTHORITY_INVALID` auf einem Rechner ohne Root-CA](docs/03-pki-tls.md#troubleshooting-err_cert_authority_invalid-auf-einem-rechner-ohne-root-ca)
- [Zammad-Passwort vergessen](docs/03-pki-tls.md#troubleshooting-zammad-passwort-vergessen)
- [Zammad-Anmeldung scheitert mit „Verifizierung des Sicherheits-Tokens fehlgeschlagen“](docs/03-pki-tls.md#troubleshooting-zammad-anmeldung-scheitert-mit-verifizierung-des-sicherheits-tokens-fehlgeschlagen)

## Host-Firewall (ufw) auf der Ticketsystem-VM

- [Drei Stolpersteine](docs/04-host-firewall.md#fehlersuche-drei-stolpersteine)

## Hardware & Speicher (Host)

- [Snapshot lässt sich nicht löschen – kein Speicher zum Zusammenführen (`E_OUTOFMEMORY`)](docs/06-hardware-speichererweiterung.md#2-problem)
