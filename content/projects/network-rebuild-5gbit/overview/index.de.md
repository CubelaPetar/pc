---
title: "Netzwerk-Umgestaltung für 5 Gbit/s"
description: "Ein Kunde hat eine symmetrische 5-Gbit/s-Leitung bestellt, und das Netzwerk musste neu gebaut werden, um sie zu bedienen: ein Sophos-Firewall-Cluster, ein Aruba-VSX-Core und LACP-Bonding auf allen Ebenen."
date: 2026-10-12
weight: 1
draft: false
categories:
  - network
tags:
  - network
  - aruba
  - aos-cx
  - vsx
  - sophos
  - firewall
  - lacp
  - migration
series:
  - Network infrastructure rebuild for a 5 Gbit/s line
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

Einer unserer Kunden wollte seinen Internetanschluss auf 5 Gbit/s aufrüsten. Die bestehende
1-Gbit/s-Leitung war ausgelastet, und man hat es gemerkt. Beide Anschlüsse sind symmetrisch in
Down- und Upload, und eine symmetrische 5-Gbit/s-Leitung bekommt man in Deutschland noch längst
nicht überall — deshalb habe ich dieses Projekt besonders gerne geleitet.

Die vorhandene Netzwerkinfrastruktur unterstützte maximal 1 Gbit/s, die neue Leitung brauchte
also ein Firewall-Cluster, das sie auch bedienen kann. Dabei habe ich gleich die gesamte
Netzwerkinfrastruktur neu aufgebaut, um mehr aus der Hardware herauszuholen, die ohnehin schon
da war: einen dedizierten WAN-Switch, ein neues Firewall-Cluster und einen richtigen Core aus
zwei Switches, die bereits im Rack standen.

Ich habe den ISP nicht um Erlaubnis gefragt, über ihn zu schreiben, deshalb wird er hier nicht
genannt. Der Kunde wird ebenfalls nicht genannt. Hostnamen, IP-Adressen und alles andere
Identifizierende wurde in der gesamten Serie geändert.

Dieser Artikel gibt einen allgemeinen Überblick über das Projekt und darüber, wie die Arbeit
strukturiert war. Es ist Teil 1 des Projekts [Neuaufbau einer Netzwerkinfrastruktur für eine 5-Gbit/s-Leitung]({{< relref "/projects/network-rebuild-5gbit" >}}).

## Das Ziel

![Das Netzwerk nach dem Umbau](after.png)

**Das WAN.** Eine einzelne Single-Mode-Faser vom ISP mit BiDi-SFP+, ein statisches /29, terminiert auf einem kleinen Aruba 6100 24G, der als reiner Layer-2-Switch arbeitet. Er existiert, damit beide Firewall-Knoten dieselbe Übergabe erreichen können. Auf der WAN-Seite gibt es keine Aggregation: ein 10-GbE-Link pro Knoten ist bereits doppelt so viel, wie die Leitung hergibt, und die Faser und der Switch sind ohnehin Single Points of Failure.

**Die Firewall.** Zwei Sophos XGS 4300 in Active-Passive-HA. Jeder Knoten erreicht den Core über eine 2 × 10-GbE-LACP-Gruppe mit je einem Link zu jedem Core-Switch, sodass ein Knoten weiterarbeitet, wenn einer der beiden Switches ausfällt. Alle internen Netze laufen als getaggte VLANs über diese Gruppe.

**Der Core.** Zwei Aruba 8360-16Y2C v2 als VSX-Paar: zwei 100-GbE-Kabel zwischen ihnen als Inter-Switch-Link und ein Keepalive über ein separates Direktkabel zwischen ihren Out-of-Band-Management-Ports. VSX ist kein Stacking. Beide Switches behalten ihre eigene Konfiguration, ihre eigene Management-Adresse und ihre eigene Control Plane. Was sie teilen, ist eine gemeinsame Identität nach unten — und genau das erlaubt es einem einzelnen Host, über beide Chassis zu bonden.

**Alles darunter.** Der Distribution-Switch erreicht den Core über ein LAG und bekommt damit einen einzelnen 20-GbE-Pfad aus zwei Uplinks, die jeweils auf einem anderen Core-Switch landen. Jeder Server und jeder Storage-Knoten macht dasselbe mit 25 GbE: ein Link zu jedem Switch, gebündelt per LACP.

Aus Sicht des Cores ist jedes einzelne davon ein Multi-Chassis-LAG, die Firewall-Gruppen eingeschlossen. Genau dafür ist das VSX-Paar da: ein LAG kann sich über beide Switches erstrecken. Das Gerät am anderen Ende konfiguriert einen ganz normalen LACP-Bond und muss nie erfahren, dass seine beiden Links in verschiedenen Chassis landen.

Eine Sache haben wir bewusst gelassen, wie sie war: Das Inter-VLAN-Routing bleibt auf der
Firewall, statt auf den Core zu wandern. Der Core würde schneller routen, aber alle Clients
hängen an 1 GbE, die Firewall ist also weit davon entfernt, der Engpass zu sein — und das
Routing dorthin zu verlagern hieße, die Inter-VLAN-Regeln der Firewall als Switch-ACLs neu zu
bauen. Sollte der Verkehr zwischen den VLANs irgendwann wirklich zum Engpass werden, ist das
der Moment, das noch einmal anzuschauen.

## Das Netzwerk vorher

![Das Netzwerk vor dem Umbau](before.png)

Die alte 1-Gbit/s-Leitung kam auf einer FritzBox an, die sie über Kupfer an die beiden Firewall-Knoten weitergab. Die Firewall war ein Paar Sophos XGS 136 in Active-Passive-HA. Das sind 1-GbE-Geräte — die neue Leitung wäre also angekommen, und nichts wäre schneller geworden. So weit war das erwartbar.

Drei weitere Dinge standen im selben Rack.

Die beiden 8360, `xyz-sw-ug-02` und `xyz-sw-ug-03`, waren für die Server und den
Storage-Cluster angeschafft worden und liefen als ganz normale Access-Switches. Jeder hat
sechzehn SFP28-Ports und zwei QSFP28-Ports, und sie waren über ein einzelnes 100-GbE-Kabel
miteinander verbunden. Das ist viel Switch für das Patchen von Servern.

`xyz-sw-ug-01`, die 6100 für die Arbeitsplätze und die Management-Ports, hatte je einen Uplink
zu beiden. Spanning Tree hat einen davon geblockt — das ist das Einzige, was Spanning Tree mit
zwei getrennten Links in zwei getrennte Bridges tun kann. Jedes Client-Paket lief also über
einen einzigen 10-GbE-Link, während der zweite als kalte Reserve danebenlag. Beide Kabel waren
längst im Rack. Niemand hatte sie gebündelt.

Dieser Block hatte einen Nebeneffekt, der später wichtig wurde. Traffic zwischen
`xyz-sw-ug-01` und `xyz-sw-ug-03` nahm den Umweg über `xyz-sw-ug-02` — über genau das
100-GbE-Kabel, das kurz darauf der VSX-Inter-Switch-Link werden sollte.

Die Server- und Storage-Bonds liefen alle im Active-Backup-Modus. Jeder hatte zwei 25-GbE-Links, einen zu jedem 8360, und benutzte immer nur einen davon. Der andere wartete auf einen Ausfall, der meistens nie kam.

## Planung und Umsetzung

Jeder Schritt wurde vorab mit dem Kunden besprochen und abgestimmt, einschließlich der Reihenfolge und der nötigen Wartungsfenster.

Wir haben die Umsetzung in zwei Phasen aufgeteilt. Die erste ist der Internetanschluss, die
Firewall und der Switch-Core: der WAN-Switch, das neue Cluster, das VSX-Paar und die
gebündelten Uplinks. Die zweite ist alles unterhalb des Cores: die Server- und Storage-Bonds
und der NDR-Sensor, von denen nichts vom ISP abhängt.

Der Kunde hat mit dem ISP einen Termin für die Aktivierung der neuen Leitung vereinbart und uns
Bescheid gegeben. Der Termin lag am Nachmittag, wir sind also am Morgen angerückt, um alles
vorzubereiten. Der ISP war außerdem bereit, die alte FritzBox-Leitung noch ein paar Tage
weiterlaufen zu lassen — damit konnten wir die neue Infrastruktur aufbauen und die alte Leitung
weiterhin als Backup-WAN-Interface nutzen.

Diese erste Phase musste an einem einzigen Tag vor Ort passieren, weil die Teile wie Zahnräder
ineinandergreifen. Die LAGs des Firewall-Clusters konnten nicht existieren, bevor es den VSX-Core gab, und der Core musste unter einer Verbindung neu gebaut werden, die weiterhin den Traffic der Firma trug.

Das hat funktioniert. Das Firewall-Cluster wurde auf die alte FritzBox-Leitung migriert, bevor der neue Anschluss überhaupt existierte — das hat ein paar Minuten Ausfall gekostet, die im Haus niemand bemerkt hat — und die neue Leitung kam Stunden später als Routing-Wechsel hoch statt als Cutover.

Der [nächste Artikel]({{< relref "/projects/network-rebuild-5gbit/wan-and-vsx" >}}) behandelt den WAN-Switch, die 5-Gbit/s-Übergabe und den Aufbau des VSX-Cores, auf dem alles andere landen musste.

## Ressourcen

Alles, was in der Serie referenziert wird, an einer Stelle. Die einzelnen Artikel verlinken
jeweils die Teilmenge, die sie tatsächlich verwenden.

**Aruba AOS-CX**

- [Aruba VSX Configuration Guide, AOS-CX 10.13](https://arubanetworking.hpe.com/techdocs/AOS-CX/10.13/PDF/vsx.pdf)
- [Aruba 8360 basic and VSX configuration (random-it-blog.de)](https://random-it-blog.de/posts/aruba-8360-basic-and-vsx-configuration-part-2/)

**Sophos Firewall**

- [XGS 4300/4500 Betriebsanleitung](https://docs.sophos.com/nsg/hardware/operatinginstructions/xgs/sophos-operating-instructions-xgs-4300-4500.pdf)
- [Backup und Restore über Modelle und Versionen hinweg](https://docs.sophos.com/nsg/sophos-firewall/21.5/Help/en-us/webhelp/onlinehelp/AdministratorHelp/BackupAndFirmware/BackupAndRestore/BackupAndRestoreModelsVersions/index.html)
- [Import und Export](https://docs.sophos.com/nsg/sophos-firewall/22.0/Help/en-us/webhelp/onlinehelp/AdministratorHelp/BackupAndFirmware/ImportExport/index.html)
- [Link aggregation groups](https://docs.sophos.com/nsg/sophos-firewall/21.5/Help/en-us/webhelp/onlinehelp/AdministratorHelp/Network/Interfaces/NetworkLinkAggregationGroups/index.html)
- [Eine Link Aggregation Group anlegen](https://docs.sophos.com/nsg/sophos-firewall/21.5/Help/en-us/webhelp/onlinehelp/AdministratorHelp/Network/Interfaces/NetworkLinkAggregationGroupAdd/index.html)
- [Ein VLAN-Interface konfigurieren](https://docs.sophos.com/nsg/sophos-firewall/20.0/Help/en-us/webhelp/onlinehelp/AdministratorHelp/Network/Interfaces/NetworkVLANInterfaceAdd/index.html)
- [Gateways](https://docs.sophos.com/nsg/sophos-firewall/22.0/Help/en-us/webhelp/onlinehelp/AdministratorHelp/Routing/Gateways/index.html)
- [Active-Passive-HA konfigurieren](https://docs.sophos.com/nsg/sophos-firewall/22.0/Help/en-us/webhelp/onlinehelp/HighAvailablityStartupGuide/HAConfiguration/HAQuickHAConfigureActivePassive/index.html)
- [HA-Voraussetzungen](https://docs.sophos.com/nsg/sophos-firewall/19.0/Help/en-us/webhelp/onlinehelp/HighAvailablityStartupGuide/AboutHA/HAPrerequisitesStartupGuide/index.html)
- [KBA-000005173: Link Aggregation in HA](https://support.sophos.com/support/s/article/KBA-000005173?language=en_US)

**Sophos NDR**

- [SPAN-Einstellungen](https://docs.sophos.com/central/ndr/help/en-us/UserGuide/settings/SPAN/)
- [NDR-Sensor auf Nutanix](https://docs.sophos.com/central/customer/help/en-us/ManageYourProducts/ThreatAnalysisCenter/Integrations/Sophos/NDR/NDRVirtual/NDRonNutanix/index.html)
- [NDR-Sensor auf ESXi und Hyper-V](https://docs.sophos.com/central/customer/help/en-us/ManageYourProducts/ThreatAnalysisCenter/Integrations/Sophos/NDR/NDRVirtual/NDRESXiHyperV/index.html)
- [SophosNDR-Proxmox (GitHub)](https://github.com/BrandonSanders48/SophosNDR-Proxmox)

**Qumulo**

- [Qumulo Administrator Guide](https://docs.qumulo.com/administrator-guide/)
- [`qq` CLI Command Guide: login](https://docs.qumulo.com/qq-cli-command-guide/login/login.html)

**Messung der Leitung**

- [librespeed-cli](https://github.com/librespeed/speedtest-cli)
- [Hetzner-Speedtest-Datei](https://nbg1-speed.hetzner.com/1GB.bin)

---

*Dieser Artikel wurde mit Unterstützung von Claude (Anthropic) verfasst.*
