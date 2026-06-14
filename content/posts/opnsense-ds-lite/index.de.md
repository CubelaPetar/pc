---
title: "OPNsense mit DS-Lite"
description: "Anleitung zur Konfiguration von OPNsense für eine DS-Lite-Verbindung, inklusive PPPoE-Setup, GIF-Tunnel und IPv6 Prefix Delegation."
date: 2026-06-05
lastmod: 2026-06-14
draft: false
categories:
  - tutorial
  - network
tags:
  - opnsense
  - firewall
  - ds-lite
  - ipv6
series:
  - Home Network with IPv6
showDate: true
showDateUpdated: true
showReadingTime: true
showTableOfContents: true
showWordCount: true
showAuthor: true
showTaxonomies: true
showPagination: true
---

## Einleitung

Ich habe zuhause eine DS-Lite-Verbindung, bereitgestellt von M-Net. Ich betreibe eine OPNsense-Firewall und musste die beiden zum Laufen bringen. Mein erster Gedanke war, meinen Internetanbieter (ISP) anzurufen und nach einem richtigen Dual-Stack-Setup mit echter öffentlicher IPv4-Adresse zu fragen — hauptsächlich wegen VPN-Zugriffsproblemen aus IPv4-only-Netzwerken. Aber ich entschloss mich, das als Herausforderung anzunehmen: ich mag IPv6, weshalb ich lernen wollte mit den Einschränkungen von DS-Lite umzugehen und Lösungen für die verbleibenden Probleme finden. Es wird einen Folgeartikel darüber geben, wie man aus einem IPv4-only-Netzwerk auf ein DS-Lite-Heimnetzwerk zugreift, indem man einen VPS als WireGuard-Hub verwendet.

OPNsense unterstützt DS-Lite, aber es erfordert eine spezifische Konfiguration und es gibt einige nicht-offensichtliche Stolperfallen. Dieser Artikel führt durch die vollständige Konfiguration.

{{< alert icon="circle-info" >}}
Diese Anleitung verwendet **M-Net** als Beispiel-ISP. M-Net ist ein deutscher regionaler Anbieter. Die VLAN-ID (40), die AFTR-Adresse und das Format des PPPoE-Benutzernamens sind M-Net-spezifisch — dein ISP kann abweichen. Das generelle Vorgehen ist für jede DS-Lite-Verbindung gleich, aber diese Werte solltest du mit deinem Anbieter abklären.
{{< /alert >}}

## Was ist DS-Lite?

DS-Lite (Dual-Stack Lite, [RFC 6333](https://datatracker.ietf.org/doc/html/rfc6333)) ist eine Übergangstechnologie, die es ISPs ermöglicht, ihren Kunden IPv6 bereitzustellen und dabei öffentliche IPv4-Adressen zu schonen. Aus Kundensicht sieht es so aus:

- Man bekommt ein **öffentliches, routbares IPv6-Prefix** (in meinem Fall ein /56 von M-Net, delegiert via DHCPv6).
- Man bekommt **keine öffentliche IPv4-Adresse**. Stattdessen wird der IPv4-Verkehr durch die IPv6-Verbindung zu einem Gerät namens **AFTR** (Address Family Transition Router) beim ISP getunnelt. Der AFTR NATet dann den Verkehr aller Kunden hinter gemeinsamen öffentlichen IPv4-Adressen (CGNAT, [RFC 6598](https://datatracker.ietf.org/doc/html/rfc6598)).

Der Tunnel zwischen dem eigenen Router und dem AFTR ist ein **GIF**-Tunnel (Generic IPv6-over-IPv6 Tunnel, genauer gesagt IPv4-in-IPv6). Innerhalb des Tunnels verwenden beide Endpunkte Adressen aus dem reservierten `192.0.0.0/29`-Block ([RFC 6333 §10](https://datatracker.ietf.org/doc/html/rfc6333#section-10)):

| Adresse | Rolle |
|---|---|
| `192.0.0.2` | Eigener Router (B4-Element) |
| `192.0.0.1` | ISP AFTR Gateway |

Die praktischen Nachteile: Man verliert eine echte öffentliche IPv4, Port-Forwarding ist nicht möglich, und VPN-Protokolle, die auf eingehende IPv4-Verbindungen angewiesen sind, benötigen eine Lösung. Jeder Host und Dienst hinter dem Router sollte im Idealfall direkt über IPv6 erreichbar sein.

## Voraussetzungen

- OPNsense (getestet auf 24.x)
- Ein DSL-Modem im Bridge-Modus — M-Net erfordert VLAN 40 auf dem WAN. Ein DrayTek Vigor 166 funktioniert gut; eine FritzBox ebenfalls.
- M-Net PPPoE-Zugangsdaten (Benutzername und Passwort aus dem Kundenportal)
- Die AFTR-IPv6-Adresse für M-Net: `2001:a60:0:2::ffff` (Hostname `aftr.prod.m-online.net` — muss selbst aufgelöst werden, OPNsense akzeptiert im GIF-Setup nur eine IP-Adresse)

## Schritt 1: Modem-Einrichtung

Das Modem muss im Bridge-Modus betrieben werden, damit OPNsense die rohen PPPoE-Frames empfängt und die Verbindung selbst verwaltet. M-Net erfordert VLAN 40 — sicherstellen, dass das Modem es korrekt durchleitet.

## Schritt 2: PPPoE-Gerät erstellen

In OPNsense: **Interfaces → Devices → Point-to-Point**.

Neuen Eintrag hinzufügen:

| Feld | Wert |
|---|---|
| Link Type | `PPPoE` |
| Link Interface(s) | Physischer WAN-Port (z.B. `igb2`) |
| Username | M-Net-Benutzername (Format: `XXXXXXX@mdsl.mnet-online.de`) |
| Password | M-Net-Passwort |
| Description | `mnet_pppoe_dslite` |

Alle anderen Felder auf den Standardwerten lassen. Speichern und übernehmen.

![PPPoE-Gerätekonfiguration in OPNsense](pppoe-device.png)

Ein neues Gerät `pppoe1` (oder `pppoe0`, wenn es das erste ist) erscheint in der Geräteliste.

## Schritt 3: WAN-Interface zuweisen und konfigurieren

Unter **Interfaces → Assignments** sicherstellen, dass das **WAN**-Interface dem neu erstellten PPPoE-Gerät (`pppoe1`) zugewiesen ist.

Dann **Interfaces → WAN** öffnen und wie folgt konfigurieren:

**IPv4 Configuration Type** → `None`
DS-Lite gibt keine echte IPv4 auf dem WAN-Interface — IPv4 kommt in einem späteren Schritt durch den GIF-Tunnel.

**IPv6 Configuration Type** → `DHCPv6`
Stellt die IPv6-Verbindung über den PPPoE-Link her und fordert die Prefix Delegation von M-Net an.

Unter **DHCPv6 client configuration** folgendes einstellen:

| Feld | Wert |
|---|---|
| Prefix Delegation Size | `56` |
| Send prefix hint | ✓ aktiviert |
| Optional Prefix ID | `9` |

Die Prefix ID bestimmt, welches Sub-Prefix des delegierten /56 das WAN-Interface selbst für seine IPv6-Adresse verwendet. Der Wert `9` reserviert ein /64 für das WAN und lässt den Rest für LAN und VLANs frei — das bedeutet auch, dass das WAN-Interface eine routbare Global Unicast Address (GUA) erhält, was für den GIF-Tunnel in Schritt 5 wichtig ist.

Speichern und übernehmen. OPNsense stellt nun die PPPoE-Verbindung her und fordert ein IPv6 /56-Prefix von M-Net an.

![WAN-Interface-Konfiguration mit DHCPv6 und Prefix Delegation](wan-interface.png)

## Schritt 4: IPv6 auf LAN und VLANs tracken

Für jedes interne Interface (LAN, VLANs) unter **Interfaces → [Interface]** folgendes einstellen:

| Feld | Wert |
|---|---|
| IPv6 Configuration Type | `Track Interface (legacy)` |
| Parent Interface | `WAN` |
| Assign Prefix ID | Ein eindeutiger Wert pro Interface (z.B. `0` für LAN, `1`–`8` für VLANs) |

Damit wird OPNsense angewiesen, die IPv6-Adresse jedes Interfaces aus dem /56-Block abzuleiten, der dem WAN delegiert wurde. Jede Prefix ID entspricht einem eigenen /64 innerhalb des /56.

![LAN-Interface konfiguriert zum Tracken von IPv6 vom WAN](lan-track-ipv6.png)

Zu diesem Zeitpunkt sollte IPv6-Konnektivität bestehen. Testen mit `ping6 2620:fe::fe` (Quad9) aus der OPNsense-Shell oder von einem Host dahinter. Wenn IPv6 funktioniert, weiter zum IPv4-Tunnel.

## Schritt 5: GIF-Tunnel erstellen

Der GIF-Tunnel transportiert IPv4-Verkehr in IPv6-Paketen zum AFTR von M-Net.

Unter **Interfaces → Devices → GIF**.

Neuen Eintrag hinzufügen:

| Feld | Wert |
|---|---|
| Local address | `WAN` (oder ein beliebiges Interface mit GUA — siehe Hinweis unten) |
| Remote address | `2001:a60:0:2::ffff` |
| Tunnel local address | `192.0.0.2` |
| Tunnel remote address | `192.0.0.1` |
| Tunnel netmask / prefix | `29` |
| Disable Ingress filtering | ✓ aktiviert |
| Description | `AFTR_MNET` |

**Welches Interface als lokale Adresse verwenden?**
Der GIF-Tunnel benötigt eine **Global Unicast Address (GUA)** als IPv6-Quelle — eine Link-Local-Adresse ist nicht routbar und erreicht den AFTR nicht. Da in Schritt 3 eine Prefix ID (`9`) auf den DHCPv6-Client des WAN gesetzt wurde, erhält das WAN-Interface selbst eine GUA aus dem delegierten /56 und funktioniert hier. Jedes andere Interface mit einer GUA (LAN, ein VLAN) funktioniert ebenfalls.

![GIF-Tunnel-Konfiguration mit Verweis auf M-Nets AFTR](gif-tunnel.png)

Speichern und übernehmen.

## Schritt 6: GIF-Interface als WANv4 zuweisen

Unter **Interfaces → Assignments** das `gif0`-Gerät als neues Interface hinzufügen. Beschreibung auf `WANv4` setzen.

**Interfaces → WANv4** öffnen:

- Interface aktivieren
- Sowohl IPv4 als auch IPv6 Configuration Type auf **None** lassen — die Adressen sind bereits in der GIF-Konfiguration definiert

Speichern und übernehmen.

![WANv4-Interface zugewiesen zu gif0 ohne IP-Konfiguration](wanv4-interface.png)

## Schritt 7: Gateway überprüfen

Wenn das WANv4-Interface zugewiesen wird, erstellt OPNsense automatisch einen Gateway-Eintrag dafür. Unter **System → Gateways → Configuration** überprüfen, ob er vorhanden ist.

Der automatisch erstellte Gateway hat das WANv4-Interface zugewiesen, ein leeres IP-Adressfeld (OPNsense leitet den Endpunkt aus der GIF-Tunnel-Konfiguration ab) und Gateway-Monitoring deaktiviert. Er muss nicht als Upstream-Gateway gesetzt werden — OPNsense routet IPv4-Verkehr automatisch durch ihn.

![Automatisch erstellter Gateway für das WANv4/GIF-Tunnel-Interface](gateway.png)

## Setup überprüfen

Von einem Host im LAN sollten nun sowohl IPv4 als auch IPv6 funktionieren:

- `curl -4 ifconfig.me` — sollte eine Adresse im Bereich `100.64.0.0/10` zurückgeben (CGNAT, normal bei DS-Lite)
- `curl -6 ifconfig.me` — sollte die öffentliche IPv6-Adresse zurückgeben

Wenn IPv4 nichts zurückgibt oder der Tunnel nicht aufgebaut wird:

1. Prüfen, ob IPv6 zuerst funktioniert (der Tunnel hängt davon ab)
2. Prüfen, ob die AFTR-Adresse korrekt ist — `aftr.prod.m-online.net` auflösen und mit `2001:a60:0:2::ffff` vergleichen
3. Prüfen, ob das GIF-Local-Address-Interface eine GUA hat (z.B. WAN mit gesetzter Prefix ID, oder LAN)

## Was nicht funktioniert

- **Port Forwarding**: Man teilt sich eine öffentliche IPv4 mit anderen Kunden hinter dem AFTR. Eingehende IPv4-Verbindungen sind nicht möglich. Die Lösung ist, IPv6 für alles zu verwenden, das von außen erreichbar sein soll, oder durch einen VPS zu tunneln — das wird im nächsten Artikel behandelt.
- **VPNs über IPv4 von außen**: Gleicher Grund. WireGuard oder OpenVPN, das auf der öffentlichen IPv6-Adresse lauscht, funktioniert einwandfrei.

## Bekannte Probleme

{{< alert icon="triangle-exclamation" cardColor="#8B1A1A" iconColor="#FF6B6B" textColor="#f1faee" >}}
Es gibt einen ungelösten Bug in OPNsense, der bei DS-Lite-Setups zu sporadischen Problemen führen kann: [opnsense/core#7713](https://github.com/opnsense/core/issues/7713). Stand OPNsense 26.1.5 ist das Issue noch offen. Wenn die IPv4-Konnektivität durch den GIF-Tunnel unerwartet abbricht oder sich inkonsistent verhält, könnte dieser Bug die Ursache sein.
{{< /alert >}}

## Ressourcen

- [DS-Lite on 23.7.6+ (OPNsense Forum)](https://forum.opnsense.org/index.php?topic=37813.0)
- [OPNsense Core Issue #7713](https://github.com/opnsense/core/issues/7713)
- [OPNsense Forum topic 46665](https://forum.opnsense.org/index.php?topic=46665.0)
- [OPNsense Forum topic 27935](https://forum.opnsense.org/index.php?topic=27935.msg136305#msg136305)
- [M-Net DS-Lite mit pfSense (cybercyber.org)](https://cybercyber.org/m-net-ds-lite-anschluss-mit-pfsense.html)
- [RFC 6333 — Dual-Stack Lite](https://datatracker.ietf.org/doc/html/rfc6333)

---

*Dieser Artikel wurde mit Unterstützung von Claude (Anthropic) verfasst.*
