---
title: "Meine Erfahrung mit Netzwerksegmentierungen"
description: "Was Netzwerksegmentierung ist, wie ich Zonen und Segmente plane und wie ein Segmentierungsprojekt in der Praxis abläuft."
date: 2026-10-11
lastmod: 2026-10-11
draft: false
categories:
  - network
tags: 
  - network
  - vlans
  - firewall
  - bsi 
  - security
series: []
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

Beruflich mache ich viele Firewall-Migrationen und Netzwerk-Umstrukturierungen für kleine und mittelständische Unternehmen, und bei fast jedem dieser Projekte kommt das Thema Netzwerksegmentierung auf. Meistens schlagen wir sie vor; nur wenige Kunden fragen selbst danach. Manche haben am Ende ein sauberes VLAN-Layout und ein striktes Regelwerk. Bei anderen entscheiden wir, die Segmentierung zu verschieben, bis die neue Firewall steht, und das ist manchmal die bessere Wahl. Was ich dabei vor allem gelernt habe: Es gibt kein Layout, das für jede Infrastruktur passt. **Welche Trennung sinnvoll ist, hängt stark von der Größe und dem Kontext des Netzwerks ab.** Ein kleines Büro, eine Schule mit Hunderten iPads und ein Gebäudeautomationsnetz mit Tausenden Controllern brauchen sehr unterschiedliche Dinge.

Meiner Meinung nach ist ein ordentlich segmentiertes Netzwerk, zusammen mit einem ordentlichen Firewall-Regelwerk, eines der wichtigsten Sicherheitsmerkmale, das eine Infrastruktur haben kann. Ich lebe und arbeite in Deutschland, wo Gesetze wie NIS2 (in Deutschland das NIS2UmsuCG) und Standards wie der BSI IT-Grundschutz oder ISO 27001 sie inzwischen mehr oder weniger verlangen. In diesem Artikel geht es aber nicht um die Vorschriften, sondern um die Praxis: Zonen und Segmente, typische Segmente, wer mit wem sprechen darf und wie ein Segmentierungsprojekt abläuft. Kundennamen und alles Identifizierende sind weggelassen.

## Was Segmentierung ist und warum

Netzwerksegmentierung bedeutet, ein Netzwerk in mehrere kleinere aufzuteilen und eine Firewall dazwischenzusetzen. Geräte, die zusammengehören, liegen im selben Segment. Alles, was von einem Segment in ein anderes will, muss durch die Firewall, und die Firewall lässt nur durch, was ausdrücklich erlaubt ist.

Der Grund ist einfach. Früher oder später wird etwas im Netzwerk kompromittiert: ein Laptop mit einem bösartigen Anhang, ein Drucker mit zehn Jahre alter Firmware, eine Kamera, von der niemand mehr weiß, wer sie gekauft hat. In einem flachen Netzwerk kann dieses Gerät mit allem anderen sprechen, auch mit dem Domain Controller, dem NAS und dem Backup-Server. In einem segmentierten Netzwerk erreicht es nur das, was sein Segment erreichen darf. Segmentierung verhindert nicht die erste Infektion. Sie begrenzt, wie weit sich ein Angreifer (oder z. B. Ransomware) danach bewegen kann, das sogenannte *Lateral Movement*, und sie verkleinert den Schadensradius. Ein Gerät sollte sich nur mit dem verbinden können, was es wirklich braucht. Das ist ein Kernprinzip von Zero-Trust.

## Zonen und Segmente

Das Dokument, auf das ich immer wieder zurückkomme, ist der IT-Grundschutz-Baustein [NET.1.1 „Netzarchitektur und -design“](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2023/09_NET_Netze_und_Kommunikation/NET_1_1_Netzarchitektur_und_design_Edition_2023.pdf?__blob=publicationFile&v=3) vom BSI (*Bundesamt für Sicherheit in der Informationstechnik*). Er ist kostenlos, nur zehn Seiten lang und sehr konkret, und vieles von dem, was folgt, basiert darauf.

Viele (mich eingeschlossen) verwenden „Netz“, „VLAN“, „Zone“ und „Segment“ mehr oder weniger austauschbar. NET.1.1 ist da strenger, und der Unterschied hilft: Eine **Zone** ist ein großer Bereich mit einem klaren Vertrauensniveau, etwa das interne Netz, die DMZ oder das Internet. Ein **Segment** ist eine Unterteilung *innerhalb* einer Zone, z. B. Clients, Server und Drucker im internen Netz, und in der Praxis meistens ein VLAN. Zonen können auch andere logische Gruppierungen von Netzen sein, etwa TRUSTED, UNTRUSTED oder WIFI. Wichtig ist, dass jedes Netz genau zu einer Zone gehört und die Zone sagt, wie sehr man ihm vertraut.

### Mit den Zonen anfangen

Bevor ich über VLANs nachdenke, denke ich über Zonen nach. Sie sind die grobe Struktur, die Segmente werden danach eingefügt. Das Modell, das ich verwende, basiert auf dem BSI (NET.1.1 und die ältere ISi-LANA-Studie) und sieht so aus:

| Zone | Vertrauen | Was dort liegt |
|---|---|---|
| Extern | nicht vertrauenswürdig | das Internet, der ISP, Partner- und Kundennetze |
| DMZ | gering | Server, die aus dem Internet erreichbar sind: Reverse Proxy, Mail-Relay, VPN-Gateway |
| Intern | vertrauenswürdig | die Segmente für Server, Clients, Drucker, Telefone, … |
| Intern, nicht vertrauenswürdig | nicht vertrauenswürdig | Gäste-WLAN, IoT, Geräte, die man nicht kontrolliert |
| Management | am sensibelsten | Management-Interfaces von Netzwerkgeräten und Hypervisoren, Monitoring, Logging, Backup |

![Die fünf Zonen und ihre Segmente, alle über die Firewall verbunden](zones.drawio.png)

Daraus ergeben sich ein paar Regeln:

- **Verbindungen werden von mehr Vertrauen zu weniger Vertrauen aufgebaut, nicht umgekehrt.** Clients dürfen Verbindungen in die DMZ öffnen. Ein Server in der DMZ öffnet nie eine Verbindung ins interne Netz. Das BSI ist hier eindeutig: Aus dem Internet oder der DMZ darf nicht auf das interne Netz zugegriffen werden.
- **Nicht vertrauenswürdige Netze werden wie das Internet behandelt.** Das BSI formuliert es genau so, und für mich ist das die nützlichste Regel überhaupt. Ein Gäste-WLAN liegt zwar physisch im Gebäude, aus Sicherheitssicht ist es aber das Internet. Es bekommt Internetzugang und sonst nichts.
- **Management ist die sensibelste Zone.** Wer die Management-Interfaces der Switches, der Firewall und der Hypervisoren erreicht, dem gehört das Netzwerk. Mehr dazu bei der Kommunikationsmatrix.

Viele Firewalls kennen Zonen als Konzept und lassen einen Regeln zwischen Zonen statt zwischen einzelnen Interfaces schreiben (bei OPNsense erfüllen Interface-Gruppen einen ähnlichen Zweck). Auch wenn deine Firewall das nicht kann: Die Zonen aufzuschreiben hält das Regelwerk konsistent.

Eine ehrliche Anmerkung: Das BSI will mindestens drei *physisch* getrennte Zonen und eine zweistufige Firewall zwischen Internet und internem Netz. In den kleinen und mittelständischen Unternehmen, mit denen ich arbeite, sehe ich das fast nie. Meistens übernimmt eine Firewall (oder ein HA-Cluster) alle Zonen, und die Trennung zwischen ihnen ist logisch. Das ist ein Kompromiss, und es ist gut zu wissen, dass es einer ist.

### VLANs trennen, Firewall-Regeln schützen

Zehn VLANs, die alle mit einer „allow any“-Regel untereinander geroutet werden, sind keine Segmentierung. Das ist ein flaches Netzwerk mit zusätzlichen Schritten. Die Sicherheit kommt aus dem Regelwerk zwischen den Segmenten, und dieses Regelwerk sollte mit *alles verbieten* beginnen und dann öffnen, was gebraucht wird. Das gilt für IPv4 und IPv6 gleichermaßen. Wer beides betreibt, braucht jede Regel für beide Protokolle, sonst steht die IPv6-Seite neben einem sorgfältig gebauten IPv4-Regelwerk sperrangelweit offen.

## Typische Segmente

Das ist das Standard-Layout, das wir bei der Arbeit verwenden. Die IDs sind nur eine Konvention, aber überall dieselben zu verwenden hilft sehr, wenn man viele Netzwerke betreut: MGMT ist bei fast jedem Kunden VLAN 50, und niemand muss darüber nachdenken.

| VLAN | Segment | Zone | Was hinein gehört |
|---|---|---|---|
| 10 | SERVER | Intern | Server, VMs, NAS, Domain Controller. Oft einfach das alte flache Netz, siehe unten. |
| 20 | CLIENT | Intern | Kabelgebundene Laptops und Desktops |
| 30 | WLAN | Intern | Firmengeräte im WLAN, mit denselben Regeln wie CLIENT |
| 40 | GUEST | Intern, nicht vertrauenswürdig | Gäste-WLAN: nur Internet, nichts Internes |
| 50 | MGMT | Management | Switches, Access Points, Firewall, Hypervisor-Management, iLO/iDRAC. VLAN 1 ist deaktiviert; die Management-Interfaces der Switches liegen hier. |
| 60 | PRINT | Intern | Drucker: alte Firmware, selten aktualisiert, aber jeder Client braucht sie |
| 70 | PHONE | Intern | IP-Telefone und Telefonanlage, oft mit eigenem Anbieter und QoS |

Bei Bedarf kommen weitere Segmente dazu:

- DMZ für Server, die aus dem Internet erreichbar sind. Nichts in der DMZ darf eine Verbindung in die internen Zonen öffnen.
- BACKUP für den Backup-Server und seinen Speicher. Ransomware geht zuerst auf die Backups, und wie das Management verbindet sich der Backup-Server zu allem, während sich nichts zu ihm verbindet. Deshalb zähle ich ihn zur Management-Zone.
- SEC für einen Schwachstellenscanner, ebenfalls in der Management-Zone (mehr dazu unten).
- IoT und Gebäudetechnik: Kameras, Smart-Home-Geräte, Gebäudeautomation, die Apple TVs in einer Schule. Geräte, die man weder patchen noch vertrauen kann, bekommen einen eigenen Käfig in der nicht vertrauenswürdigen Zone.

In größeren Netzwerken bekommen DNS, DHCP und die Domain Controller ein eigenes INFRA-Segment, wie es das BSI vorschlägt. In den kleinen und mittelständischen Büros, mit denen ich arbeite, steht der Domain Controller meistens im SERVER-Segment, und DHCP macht oft die Firewall.

Ein paar Dinge möchte ich hervorheben:

- Der Hypervisor taucht zweimal auf. Sein Management-Interface (die Proxmox- oder vCenter-Oberfläche, das iLO) gehört ins MGMT, seine VMs ins SERVER-Segment oder wo sie sonst hinpassen. Sonst wird der Hypervisor zur Brücke zwischen den Segmenten, genau davor warnt das BSI.
- DMZ ist nicht GUEST. Die DMZ ist für *Server*, die man ins Internet stellt. Nicht vertrauenswürdige *Clients* gehören ins GUEST- oder ins IoT-Segment. Niemand steckt einen Laptop in die DMZ.
- Man braucht nicht alle. Grundsätzlich könnte jede Gerätegruppe ein eigenes Netz bekommen, aber jedes zusätzliche Segment bedeutet mehr Regeln, mehr DHCP-Bereiche und mehr Dinge, die kaputtgehen. Eine Handvoll Segmente mit einem strengen Regelwerk ist besser als zwanzig mit einem schlampigen.

## Die Kommunikationsmatrix

Die Tabelle der Segmente ist nur die halbe Miete. Die andere Hälfte ist die Frage, wer mit wem sprechen darf. Ich schreibe das als Matrix auf: Zeilen sind die Quelle, Spalten das Ziel, und jede Zelle, die nicht ausdrücklich erlaubt ist, ist verboten.

| von \ nach | MGMT | SERVER | CLIENT / WLAN | PRINT | PHONE | BACKUP | SEC | IOT | GUEST | DMZ | Internet |
|---|---|---|---|---|---|---|---|---|---|---|---|
| **MGMT** | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | – | ✔ | nur Updates |
| **SERVER** | – | ✔ | – | Druck-Ports | – | – | – | – | – | – | eingeschränkt |
| **CLIENT / WLAN** | – | DNS, AD, App-Ports | ✔ | Druck-Ports | – | – | – | gezielt | – | gezielt | ✔ |
| **PRINT** | – | DNS, Scan-to-Share | – | ✔ | – | – | – | – | – | – | – |
| **PHONE** | – | – | – | – | ✔ | – | – | – | – | – | nur SIP-Anbieter |
| **BACKUP** | Hypervisor-API | Pull | – | – | – | ✔ | – | – | – | – | – |
| **SEC** | Scan | Scan | Scan | Scan | Scan | Scan | ✔ | Scan | – | Scan | HTTPS-Updates |
| **IOT** | – | – | – | – | – | – | – | ✔ | – | – | nur Hersteller-Cloud |
| **GUEST** | – | – | – | – | – | – | – | – | ✔ | – | ✔ |
| **DMZ** | – | – | – | – | – | – | – | – | – | ✔ | ✔ |

Hinter dieser Matrix stecken ein paar Regeln:

**Das Management verbindet sich nach außen, nichts verbindet sich hinein.** MGMT darf jedes Segment erreichen, das es verwaltet. Nichts aus einer anderen Zone baut eine Verbindung ins MGMT auf, außer einer Admin-Workstation oder einem Jump Host, der selbst in einem geschützten Segment steht. BACKUP und SEC liegen selbst in der Management-Zone: Wenn der Backup-Server VM-Backups über das Management-Interface des Hypervisors zieht, bleibt dieser Verkehr innerhalb der Zone. Die Management-Anforderungen des BSI gehen noch weiter und teilen den Management-Bereich in mehrere eigene Segmente auf, was für ein kleines Büro etwas viel ist.

**Der Backup-Server holt sich die Daten.** Produktivsysteme sollten nicht auf den Backup-Speicher schreiben oder dort etwas löschen können. Der Backup-Server verbindet sich zu den Servern (und für VM-Backups zum Hypervisor) und zieht die Daten (Pull). Wenn Ransomware den Fileserver verschlüsselt, kommt sie von dort nicht an die Backups.

**Auch ausgehender Verkehr zählt.** Bei der Matrix geht es nicht nur um eingehenden Verkehr. IoT-Geräte brauchen nur die Cloud ihres Herstellers, Drucker brauchen überhaupt kein Internet, und DNS-Anfragen von Clients sollten an den eigenen Resolver gehen, nicht an das, was im Gerät fest eingetragen ist.

Ein echtes Beispiel: Bei mehreren Kunden betreiben wir einen Schwachstellenscanner als kleine VM in einem eigenen SEC-Segment. Die Regeln dafür sind kurz: SEC zu allen Segmenten außer GUEST erlaubt Ping und die Dienste, die der Scanner braucht, SEC ins Internet erlaubt nur HTTPS für Updates, und alles in Richtung SEC ist verboten. Sauberer wird eine Zeile in der Matrix kaum.

## Wie ein Segmentierungsprojekt abläuft

VLANs zu entwerfen ist der spaßige Teil. Geräte umzuziehen ist das, wo die Zeit hingeht. Mein größtes Segmentierungsprojekt war ein Bürogebäude mit vier Stockwerken, rund zehn Räumen pro Stockwerk, einem Switch pro Stockwerk und fast in jedem Raum einem Drucker. Das Ziel waren MGMT-, SERVER-, CLIENT- und PRINTER-Netze.

**Die Bestandsaufnahme ist der wichtigste Teil.** Entweder man ist gut vorbereitet und hat (fast) jedes Gerät samt dem Switch-Port, an dem es hängt, dokumentiert, oder man verbringt Stunden damit, durchs Gebäude zu laufen und herauszufinden, welches Gerät an welchem Port steckt. In diesem Projekt haben wir zuerst jedes Gerät mit MAC-Adresse, IP-Adresse und Switch-Port dokumentiert und uns die Gebäudepläne besorgt. Bei einem anderen Kunden stellte sich der „Core-Switch“ als unmanaged Switch in der Küche heraus, und niemand wusste, wie die Verkabelung lief. So ein Projekt beginnt mit der Bestandsaufnahme, nicht mit VLANs. Ein Artikel des CMU Software Engineering Institute (SEI) hat hier einen guten Punkt: *ausreichende* Informationen anstreben, nicht perfekte, sonst fängt man nie an.

Dann kommt der Plan: die Geräte einordnen, Zonen und Segmente festlegen und die Kommunikationsmatrix schreiben. Mit den kritischsten Systemen anfangen, statt alles auf einmal umbauen zu wollen.

Ein praktischer Tipp: **die Regeln zuerst auf Aliasen aufbauen.** Bevor wir ein einziges Gerät umgezogen haben, haben wir Aliase für „Clients“, „Drucker“, „Fileserver“ und so weiter angelegt und alle neuen Regeln dagegen geschrieben, solange noch alles im flachen Netz lag. Als die Geräte in ihre neuen VLANs umgezogen sind, mussten wir nur den Inhalt der Aliase ändern, nicht die Regeln.

Was wir beim Umzug gemacht haben und was ich wieder so machen würde:

- **Das alte flache Netz als Server-VLAN behalten.** Wir haben es einfach umbenannt. Die Server haben ihre IP-Adressen behalten, das erspart viel Ärger mit fest eingetragenen IPs, DNS-Einträgen und an Adressen gebundenen Lizenzen. Diesen Trick habe ich seitdem bei fast jedem Kunden angewendet.
- **Clients sind einfach.** Sie bekommen ihre Adresse per DHCP, also ändert man das VLAN am Switch-Port, und sie kommen im neuen Netz wieder hoch.
- **Drucker sind der schwierigste Teil.** Jeder Drucker hat eine neue IP und einen neuen Hostnamen bekommen und musste auf dem Druckserver angepasst (oder neu angelegt) werden. Wir haben ihnen ein Namensschema mit Stockwerk und Nummer im Hostnamen gegeben, mit denselben Ziffern im letzten Oktett der IP (`…-og3-04` bekommt `.34`), sodass man die Adresse am Etikett ablesen kann.
- **Jede Netzwerkdose beschriften.** Wir haben alle beschriftet: der erste Port in jedem Raum `PRT`, die anderen `CLT`, und `MGMT`, wo ein Access Point oder ein Managed Switch dranhängt. Klingt langweilig, aber sechs Monate später muss niemand raten. Im Nachhinein hätte ich auf jedes Dosen-Etikett auch den Switch- bzw. Patchpanel-Port geschrieben.
- **In Phasen vorgehen.** Zuerst ein Pilot mit ein paar Clients und einem Drucker, dann alle Clients, dann die Drucker Stockwerk für Stockwerk.
- **Früh an Broadcast und Multicast denken.** Viele Broadcast- und Multicast-Protokolle (mDNS/Bonjour, SSDP, NetBIOS, DHCP) funktionieren nur innerhalb eines Segments, und das bereitet manchen echtes Kopfzerbrechen :-). Sobald Drucker und MacBooks in unterschiedlichen VLANs liegen, finden die MacBooks die Drucker nicht mehr. DHCP ist einfach: Die Firewall stellt es auf jedem VLAN bereit, oder man nimmt ein Relay. Für mDNS habe ich Reflectors eingesetzt, etwa eine Avahi-VM, damit Lehrkräfte und Schüler an einer Schule ihre Apple TVs finden, oder das mDNS-Repeater-Plugin von OPNsense für Drucker. Ein Reflector reicht aber *alle* angekündigten Dienste zwischen den VLANs durch und reißt damit ein Loch in die Segmentierung. Bei einem kleinen Büro, das nur mit Macs arbeitet, haben wir die Drucker einfach im Client-VLAN gelassen.

Nach dem Umzug ist die Arbeit nicht vorbei:

- Testen: aus jedem Segment versuchen, jedes andere Segment zu erreichen (`nmap` reicht), und prüfen, dass nur das funktioniert, was die Matrix erlaubt.
- Loggen: verworfenen Verkehr zwischen Segmenten protokollieren und tatsächlich ansehen. Das zeigt Fehlkonfigurationen und manchmal Dinge, die gar nicht da sein sollten.
- Überprüfen: regelmäßig, mindestens einmal im Jahr und nach jeder größeren Änderung.

Den Schritt nach dem Projekt vergessen viele: Jeder neue Server, Drucker oder Access Point muss im richtigen Segment landen, und für jede neue Anwendung müssen ihre Verbindungen in die Matrix. Wenn das nicht Teil des normalen Change-Prozesses ist, wird das Netzwerk langsam wieder flach. Und so langweilig es klingt: Es braucht Rückendeckung von der Geschäftsführung. Segmentierung verändert, wie Leute arbeiten (der Drucker ist plötzlich weg, das NAS wird nicht mehr gefunden), und ohne diese Rückendeckung endet die erste Beschwerde mit einer „allow any“-Regel.

Und manchmal ist die richtige Entscheidung, es (noch) nicht zu machen. Ein Kunde wollte zuerst die neue Firewall und hat die Segmentierung verschoben. Bei einem anderen hatten wir geplant, ein altes Netz in ein VLAN umzuziehen, und haben die Idee verworfen, weil der Aufwand größer war als der Nutzen. Das ist in Ordnung, solange es eine bewusste Entscheidung ist und irgendwo dokumentiert wird.

## Fazit

Technisch gesehen ist Netzwerksegmentierung ein paar VLANs, eine Firewall dazwischen und ein Regelwerk, das mit „alles verbieten“ anfängt. Die Konfiguration dauert einen Nachmittag. Die Planung, die Dokumentation und das Umziehen der Geräte dauern Wochen.

Mein Rat: zuerst die Zonen, dann die Segmente. Management, Server, Clients und Gäste trennen. Aufschreiben, wer mit wem sprechen darf. Testen. In die Logs schauen. Erweitern, wenn man weiß, wo die echten Risiken liegen, statt am ersten Tag zwanzig Segmente zu entwerfen. Und wer eine Referenz sucht: BSI NET.1.1 lesen. Sind nur zehn Seiten.

Und noch eine letzte Sache, weil „Zero-Trust“ in jeder Hersteller-Präsentation vorkommt: Ich sehe Segmentierung als Voraussetzung für Zero-Trust, nicht als etwas, das dadurch ersetzt wird. Man muss seine Segmente und Verbindungen kennen, bevor man anfangen kann, pro Benutzer und pro Gerät zu entscheiden.

## Quellen

**Offizielle Dokumente**

- [BSI IT-Grundschutz NET.1.1 Netzarchitektur und -design (Edition 2023)](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2023/09_NET_Netze_und_Kommunikation/NET_1_1_Netzarchitektur_und_design_Edition_2023.pdf?__blob=publicationFile&v=3)
- [BSI ISi-L: Secure Connection of Local Networks to the Internet (2008, englisch)](https://www.bsi.bund.de/SharedDocs/Downloads/EN/BSI/InternetSecurity/ISi-LANA-ISi-L.pdf?__blob=publicationFile&v=1), alt, aber die Prinzipien gelten noch
- [NIST SP 800-207 Zero Trust Architecture (englisch)](https://csrc.nist.gov/pubs/sp/800/207/final)

**Artikel**

- [CMU SEI: Network Segmentation: Concepts and Practices (2020, englisch)](https://www.sei.cmu.edu/blog/network-segmentation-concepts-and-practices/)

---

*Dieser Artikel wurde mit Unterstützung von Claude (Anthropic) verfasst.*
