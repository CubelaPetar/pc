---
title: "Der WAN-Switch und der VSX-Core"
description: "Eine 5-Gbit/s-Übergabe auf einem Switch statt auf einem Firewall-Port terminieren, und zwei Aruba 8360 von Access-Switches zu einem VSX-Paar umbauen, ohne das Netzwerk lahmzulegen."
date: 2026-10-12
weight: 2
draft: false
categories:
  - network
tags:
  - network
  - aruba
  - aos-cx
  - vsx
  - lacp
  - spanning-tree
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

Die neue Leitung brauchte einen Endpunkt, deshalb wurde der WAN-Switch schon Tage vorher im
Büro konfiguriert. Wir haben ihm eine öffentliche Adresse gegeben. Sobald die Faser eingesteckt
war, konnten wir die Leitung also direkt vom Switch aus testen, statt darauf zu warten, dass die
Firewall dahinter bereit ist.

Das Erste, was wir vor Ort brauchten, war ein funktionierender VSX-Core, damit die bereits auf
dem Firewall-Cluster vorbereiteten LAG-Interfaces eine Gegenstelle hatten.

Diese beiden Komponenten wurden am Tag selbst als Erste eingebaut und konfiguriert, deshalb
kommen sie hier auch zuerst. Dieser Artikel behandelt den WAN-Switch, die Details des neuen
Anschlusses und den Aufbau des VSX-Cores. Es ist Teil 2 des Projekts [Neuaufbau einer Netzwerkinfrastruktur für eine 5-Gbit/s-Leitung]({{< relref "/projects/network-rebuild-5gbit" >}}).

![Der Teil des Netzwerks, um den es in diesem Artikel geht](infra_core.drawio.png)


## Der neue Anschluss

Der ISP liefert die Leitung auf einer einzelnen Singlemode-Faser mit einem BiDi-SFP+ und
übergibt ein statisches /29. Beide Richtungen laufen auf derselben Faser mit unterschiedlichen
Wellenlängen, deshalb gibt es eine Faser statt des üblichen Paars.

Ein /29 umfasst acht Adressen, davon sechs nutzbar. Eine dieser sechs ist das Gateway des ISP,
bleiben fünf für uns:

| Adresse | Rolle |
|---|---|
| `203.0.113.88/29` | Übergabenetz |
| `203.0.113.89` | ISP-Gateway |
| `203.0.113.90` | Firewall-WAN, geteilt von beiden HA-Nodes |
| `203.0.113.91` | frei |
| `203.0.113.92` | frei |
| `203.0.113.93` | frei |
| `203.0.113.94` | Management des WAN-Switches |
| `203.0.113.95` | Broadcast |

Das Cluster braucht nur eine WAN-Adresse, weil es aktiv-passiv läuft. Der passive Node hat auf diesem Interface überhaupt keine Adresse und übernimmt `.90`, sobald er aktiv wird.

Die Faser geht nicht in die Firewall. Sie geht in einen kleinen Aruba 6100 24G, und beide Firewall-Nodes hängen an diesem. Der Grund ist das HA-Paar: Ein Kabel lässt sich nur in ein Gerät stecken. Ein einfaches Layer-2-Segment vor dem Paar gibt beiden Nodes dieselbe Übergabe.

Auf der WAN-Seite gibt es keine Aggregation. Ein 10-GbE-Link pro Node ist bereits doppelt so viel wie die Kapazität einer 5-Gbit/s-Leitung, und Faser und Switch sind ohnehin Single Points of Failure.

Beide Firewalls und den WAN-Switch haben wir vorab im Büro mit ihren echten Adressen konfiguriert, das /29 wurde dabei in einer Laborumgebung simuliert. Am Tag selbst war die WAN-Seite dann nur noch ein Kabel und keine Konfigurationsänderung.

## Der WAN-Switch

Es ist ein Aruba 6100 24G 4SFP+, so dumm konfiguriert wie möglich. Spanning Tree ist aus, weil der Switch keinen redundanten Pfad hat und damit auch keine Schleife, die aufzulösen wäre. Dass die ungenutzten Ports abgeschaltet sind, verhindert, dass später jemand durch Fehlverkabelung eine Schleife baut.

Der Switch sitzt außerhalb der Firewall und hat keinen Out-of-Band-Management-Port. Die einzige Adresse, die er haben kann, ist eine aus dem öffentlichen /29. Der ursprüngliche Plan war, ihn über die Konsole zu konfigurieren und die Adresse danach wieder zu entfernen, sodass er gar kein Management-Interface mehr hat.

Dieser Plan ging nicht auf. Die Idee war, seinen Konsolenport mit einem Server im Rack zu verbinden und diesen Server als serielle Brücke zu betreiben, damit die Konsole wenigstens über das Netzwerk erreichbar ist. Der 6100 hat USB-C als Konsolenanschluss, ich hatte ein USB-C-auf-USB-C-Kabel dabei, und kein Server im Rack hatte einen Port, in den dieses Kabel gepasst hätte.

Damit der Switch überhaupt über das Netzwerk erreichbar bleibt, haben wir die öffentliche Adresse `203.0.113.94` behalten und stattdessen gehärtet. SSH funktioniert nur mit Schlüssel, Passwort-Authentifizierung ist deaktiviert, es gibt kein Webinterface, und eine Control-Plane-ACL beschränkt Port 22 auf die öffentliche Adresse unseres Büros.

Das bleibt ein Management-Interface auf einer öffentlichen Adresse, und mir wäre es lieber, es wäre keins. Es sammelt Brute-Force-Versuche wie alles mit einem offenen Port, und die ACL ist das Einzige, was davor steht. Ein Out-of-Band-Port würde das lösen. Den hat der Switch nicht, aber ein Konsolenserver im Rack würde es ebenfalls lösen, und das ist die Lösung, auf die es sich zurückzukommen lohnt.

Eine Falle beim Schreiben einer solchen ACL: Eine AOS-CX-Access-List hat am Ende ein implizites Deny. Eine Liste, die nicht mit einem expliziten Permit endet, nimmt dem Switch also neben dem SSH-Zugang aller anderen auch sein eigenes ICMP und Routing weg. Vor dem Speichern über die Konsole testen.

## VSX ist kein Stacking

Die beiden Aruba 8360-16Y2C v2 standen bereits im Rack und arbeiteten als Access-Switches. Beim Neuaufbau werden sie zu einem VSX-Paar.

Das Erste, was man verstehen muss, ist, was VSX nicht tut. Es ist kein Stacking. Es gibt keine gemeinsame Control Plane und keine gemeinsame Konfiguration. Beide Switches behalten ihre eigene Konfiguration, ihre eigene Management-Adresse und ihre eigene Kopie von allem. Nichts, was man auf dem einen eingibt, erscheint auf dem anderen.

Was sie teilen, ist eine Identität gegenüber allem, was unter ihnen hängt. Sie geben eine gemeinsame `system-mac` bekannt, also eine Spanning-Tree-Bridge-ID, und sie können eine Link Aggregation Group terminieren, deren zwei Member-Links in unterschiedlichen Chassis landen. Genau darum geht es. Ein Server mit je einem Kabel zu jedem Switch betreibt einen ganz normalen LACP-Bond und weiß nicht, dass seine beiden Links auf unterschiedlicher Hardware enden.

## Die LAGs

In dieser Phase kommen vier Link Aggregation Groups dazu. Die Nummerierung ist willkürlich, aber
bewusst gewählt: Das Netz ist in Zehnerschritten organisiert, eine LAG-Nummer verrät also, wofür
sie da ist.

| LAG | Zweck | `xyz-sw-ug-02` | `xyz-sw-ug-03` | `xyz-sw-ug-01` |
|---|---|---|---|---|
| 128 | VSX-ISL | `1/1/17`, `1/1/18` | `1/1/17`, `1/1/18` | — |
| 21 | Uplink zum Distribution-Switch | `1/1/16` | `1/1/16` | `1/1/51`, `1/1/52` |
| 11 | Firewall-Node A | `1/1/15` | `1/1/15` | — |
| 12 | Firewall-Node B | `1/1/7` | `1/1/7` | — |

LAG 11, 12 und 21 sind auf den 8360 Multi-Chassis-LAGs. Auf dem Distribution-Switch ist
`lag 21` eine normale LAG, weil dieser Switch nicht weiß, dass er mit zwei Chassis spricht, und
es auch nicht wissen muss. Die beiden Enden derselben Aggregation sind unterschiedlich
konfiguriert, und das ist normal.

Beim Anlegen gibt es zwei Dinge, die man leicht falsch macht.

**Sie von Anfang an als `multi-chassis` anlegen.** Eine AOS-CX-LAG lässt sich nachträglich nicht
in eine Multi-Chassis-LAG umwandeln. Der Switch akzeptiert `multi-chassis` auch ganz ohne
VSX-Konfiguration, es spricht also nichts dagegen. Wenn eine LAG jemals beide Member überspannen
könnte, sollte man sie jetzt so deklarieren, sonst baut man sie später neu.

**Nichts wird synchronisiert.** Jeder Member braucht seinen eigenen Block
`interface lag N multi-chassis` und seine eigene Zuordnung der Member-Ports, auf beiden von Hand
eingetippt. Außerdem weigert sich ein Port, einer LAG beizutreten, solange er noch eine eigene
VLAN-Konfiguration hat. Die muss also zuerst weg.

Das Hinzufügen der Member zu `lag 21` ist der Schritt, der Live-Traffic verschiebt, deshalb
passiert er Link für Link. Nimmt man auf der Core-Seite beide Uplinks in die LAG auf, während
der Distribution-Switch noch zwei einfache Ports hat, schneidet man den Core von allem ab,
einschließlich der Session, in der man gerade tippt.

## Aufbau des VSX-Paars

Der Inter-Switch-Link besteht aus zwei 100-GbE-Kabeln in `lag 128`, was auf jedem Member beide
QSFP28-Ports belegt. Das Keepalive läuft separat: ein einfaches Kupferkabel zwischen den beiden
Out-of-Band-Management-Ports, das ein /30 im Management-VRF trägt.

Das Keepalive ist dazu da, ein Patt aufzulösen. Fällt der ISL weg, muss jeder Member wissen, ob
sein Peer tot oder nur nicht erreichbar ist, und das Keepalive ist der Weg, das zu unterscheiden.
Da es ein anderer physischer Pfad ist als der ISL, fällt es unabhängig davon aus.

```
! beide Member
interface 1/1/17,1/1/18
    no routing
    lag 128
    no shutdown

! xyz-sw-ug-02
vsx
    system-mac 02:00:00:00:01:00
    inter-switch-link lag 128
    role primary
    keepalive peer 10.99.99.2 source 10.99.99.1 vrf mgmt

! xyz-sw-ug-03 — gleiche system-mac, Rolle secondary, Keepalive umgekehrt
```

Vor dem Anwenden des `vsx`-Blocks über das Keepalive pingen. VSX bildet sich nicht über ein
Keepalive, das keinen Traffic durchlässt, und am Tag selbst war dieses Kabel das einzige neue
in diesem Schritt.

## Die Spanning-Tree-Priorität

Sobald VSX läuft, geben beide Member dieselbe Bridge-MAC bekannt. Sie müssen deshalb auch
dieselbe Priorität bekanntgeben, sonst sieht Spanning Tree eine Bridge, die sich selbst
widerspricht.

Die Reihenfolge ist wichtig, und ein Fehler ist teuer. Setzt man die Priorität auf dem zweiten
Switch, *bevor* VSX läuft, bekommt er eine bessere Bridge-ID als der erste, weil seine eigene
MAC die niedrigere der beiden ist. Er übernimmt die Root-Rolle, und das gesamte Netz
konvergiert neu um ihn herum. Die Priorität wird also gesetzt, nachdem das Paar gebildet ist
und sich eine MAC teilt, niemals vorher.

```
xyz-sw-ug-03(config)# spanning-tree priority 4
```

## Die Konfiguration prüfen

```
xyz-sw-ug-02# show vsx status
VSX Operational State
---------------------
  ISL channel             : In-Sync
  ISL mgmt channel        : operational
  Config Sync Status      : In-Sync
  NAE                     : peer_reachable
  HTTPS Server            : peer_reachable

Attribute           Local               Peer
------------        --------            --------
ISL link            lag128              lag128
ISL version         2                   2
System MAC          02:00:00:00:01:00   02:00:00:00:01:00
Platform            8360                8360
Software Version    LL.10.10.1030       LL.10.10.1030
Device Role         primary             secondary
```

Beide Member laufen mit derselben Softwareversion, was VSX voraussetzt.

In der LACP-Ansicht wird das Design sichtbar. Die Ausgabe ist auf die vier LAGs von oben gekürzt:

```
xyz-sw-ug-02# show lacp interfaces

Actor details of all interfaces:
----------------------------------------------------------------------------------
Intf       Aggr       Port  Port  State   System-ID         System Aggr Forwarding
           Name       Id    Pri                             Pri    Key  State
----------------------------------------------------------------------------------
1/1/15     lag11(mc)  15    1     ALFNCD  02:00:00:00:01:00 65534  11   up
1/1/7      lag12(mc)  7     1     ALFNCD  02:00:00:00:01:00 65534  12   up
1/1/16     lag21(mc)  16    1     ALFNCD  02:00:00:00:01:00 65534  21   up
1/1/17     lag128     18    1     ALFNCD  aa:bb:cc:00:00:02 65534  128  up
1/1/18     lag128     19    1     ALFNCD  aa:bb:cc:00:00:02 65534  128  up
```

`ALFNCD` ist der gesunde Zustand: active, aggregable, in sync, collecting und distributing.

Die interessante Spalte ist die System-ID. Die drei Multi-Chassis-LAGs zeigen
`02:00:00:00:01:00`, die gemeinsame VSX-System-MAC, weil sie für das Gerät am anderen Ende von
einem logischen Switch kommen. Der ISL zeigt `aa:bb:cc:00:00:02`, die eigene MAC dieses
Switches, weil der ISL eine gewöhnliche LAG zwischen zwei Nachbarn ist und nichts, was beide
überspannt. In diesem Unterschied steckt die ganze Architektur in einer Spalte.

Spanning Tree sieht es genauso:

```
xyz-sw-ug-02# show spanning-tree
Spanning tree status      : Enabled Protocol: MSTP

MST0
  Root ID    Priority   : 16384
             MAC-Address: 02:00:00:00:01:00
             This bridge is the root

  Bridge ID  Priority  : 16384
             MAC-Address: 02:00:00:00:01:00

Port         Role           State      Cost    Priority  Type
------------ -------------- ---------- ------- --------- ----------
lag11        Designated     Forwarding 2000    64        P2P
lag12        Designated     Forwarding 2000    64        P2P
lag21        Designated     Forwarding 2000    64        P2P
```

Beide Switches melden das, und beide behaupten, Root zu sein, weil sie für Spanning Tree eine
einzige Bridge sind.

## Der blockierte Uplink

Vor dem Neuaufbau hatte der Distribution-Switch zwei Uplinks, je einen zu jedem 8360. Spanning
Tree hat einen davon blockiert, was das Einzige ist, was es mit zwei separaten Links zu zwei
separaten Bridges tun kann. Jedes Client-Paket lief über einen einzigen 10-GbE-Link, während der
zweite als kalte Reserve danebenlag.

Sobald beide Uplinks Member von `lag 21` sind, sieht Spanning Tree einen logischen Port statt
zweier konkurrierender, und die Blockierung verschwindet. Die einzelnen Ports tauchen in der
Spanning-Tree-Tabelle gar nicht mehr auf, was oben zu sehen ist: Es gibt eine Zeile `lag21` und
keine Zeile `1/1/16`.

Das Ergebnis sind 20 GbE aktiv-aktiv und ein Pfad, der den Ausfall eines der beiden Core-Switches
ganz ohne Rekonvergenz übersteht statt mit einer.

## Die Core-Switches synchronisieren

Weil beide Mitglieder eigene Konfigurationen behalten, muss alles, was übereinstimmen soll,
zweimal getippt werden. Am Tag des Aufbaus ist das kein Problem, danach ist es ein langsames
Leck, denn Drift bleibt unsichtbar.

Angenommen, ein VLAN ist auf `lag 11` auf einem Mitglied erlaubt und auf dem anderen nicht. Der
Link bleibt oben und LACP bleibt zufrieden, der Bond schaltet also nie um. Frames laufen auf
dem Bein, das der Hash gerade wählt, einfach ins Leere, und es sieht nach einem
Anwendungsproblem aus statt nach einem Netzwerkproblem.

Konfigurationssynchronisation schließt diese Lücke. Der Primary schiebt alles, was markiert
ist, zum Secondary, dessen Kopie damit verwaltet statt eigenständig editierbar wird. Objekte
werden einzeln markiert:

```
! nur auf dem Primary
vlan 10
    vsx-sync

interface lag 11
    vsx-sync vlans
```

und ganze Feature-Kategorien werden im `vsx`-Kontext aufgeführt:

```
vsx
    vsx-sync lldp snmp ssh stp-global time
```

Jedes VLAN und jedes Multi-Chassis-LAG trägt eine Markierung. Der ISL nicht, denn er ist kein
Multi-Chassis-LAG.

Zwei Dinge sollte man vorher wissen. Die Markierungen gehören **nur auf den Primary**; dasselbe
Objekt auf beiden Mitgliedern zu markieren, erzeugt genau den Sync-Konflikt. Und das Aktivieren
macht den Primary maßgeblich: Unterscheiden sich die beiden in etwas, das gerade synchronisiert
wird, wird die Version des Secondary ersetzt. Erst die beiden Konfigurationen vergleichen,
echte Unterschiede von Hand auflösen, und dann ändert das Einschalten am Tag selbst gar nichts
mehr. Genau das ist das Ziel.

Eines macht der Sync nicht: Er überträgt keine Zuordnung von physischen Ports zu LAGs. Welcher
Port zu welchem LAG gehört, bleibt per Design pro Mitglied, weshalb die Handarbeit weiter oben
in diesem Artikel Handarbeit bleibt.

`show vsx status` ist dafür nicht die richtige Prüfung. Der Befehl zeigt den Zustand von VSX
selbst und meldet In-Sync, ganz gleich ob die beiden Konfigurationen übereinstimmen. Der
Befehl, der sie tatsächlich vergleicht, ist:

```
xyz-sw-ug-02# show vsx config-consistency
Configurations              Local                              Peer
------------------          ------                             ------
Software Version            LL.10.10.1030                      LL.10.10.1030
System MAC                  02:00:00:00:01:00                  02:00:00:00:01:00
System Profile              Aggregation-Leaf                   Aggregation-Leaf

VSX VLAN List
-------------
Local ISL VLANs : 1 10 20 30 40 50 60 70 80
Peer ISL VLANs  : 1 10 20 30 40 50 60 70 80

STP Configurations          Local                              Peer
------------------          ------                             ------
STP Mode                    mstp                               mstp
MST Config Digest           AC36177F50283CD4B83821D8AB26DE62   AC36177F50283CD4B83821D8AB26DE62
```

Die Zeile mit dem MST Config Digest ist die nützliche. Sie ist ein Hash der gesamten
Spanning-Tree-Region-Konfiguration, zwei Switches mit gleichem Digest stimmen also wirklich
überein und sehen sich nicht nur ähnlich.

## Anhang: detaillierte Konfiguration

Alles von oben, in der Reihenfolge, in der es tatsächlich eingegeben wurde. Portnummern und
VLAN-IDs sind spezifisch für dieses Netz, also lesen statt einfügen.

### Der WAN-Switch

Über die Konsole, bevor er ins Rack kommt.

```
interface 1/1/1-1/1/24
    shutdown

interface vlan 1
    ip address 203.0.113.94/29

ip route 0.0.0.0/0 203.0.113.89

no spanning-tree

no ssh password-authentication
user admin authorized-key <key>
ssh maximum-auth-attempts 3
ssh login-grace-time 30

apply access-list ip <name> control-plane vrf default
```

Den Shutdown-Bereich so anpassen, dass der Faser-Port und die beiden Firewall-Ports aktiv
bleiben. Der Webserver wird nie eingeschaltet. Die Control-Plane-ACL mit einem expliziten Permit
am Ende schreiben und vor dem Speichern über die Konsole testen.

### Vorbereitung am Vorabend

Nichts davon verschiebt Traffic. Die LAG-Interfaces werden leer angelegt, und die Member-Ports
werden im Zustand `shutdown` hinzugefügt.

```
! xyz-sw-ug-02 und xyz-sw-ug-03

interface lag 11 multi-chassis
    no shutdown
    no routing
    lacp mode active
    vlan trunk native 1
    vlan trunk allowed 10,20,30,40,50,60,70,80

interface lag 12 multi-chassis
    no shutdown
    no routing
    lacp mode active
    vlan trunk native 1
    vlan trunk allowed 10,20,30,40,50,60,70,80

interface 1/1/15
    shutdown
    no vlan access 60
    no routing
    lag 11

interface 1/1/7
    shutdown
    no vlan access 60
    no routing
    lag 12

interface lag 128
    no shutdown
    no routing
    lacp mode active
    vlan trunk native 50
    vlan trunk allowed all

interface lag 21 multi-chassis
    no shutdown
    no routing
    lacp mode active
    vlan trunk native 50
    vlan trunk allowed all
```

Native VLAN 1 auf den beiden Firewall-LAGs ist Absicht und unterscheidet sich von jedem anderen
Trunk hier. Die Firewall-Seite dieser Aggregation hat ihre eigene untagged Adresse, und VLAN 1
steht nicht in der Liste der erlaubten VLANs, diese Frames gehen also nirgendwohin. Nicht zu
native 50 „aufräumen“.

Die Keepalive-Adressen kommen auf die Out-of-Band-Ports, die sonst keine Adresse haben:

```
! xyz-sw-ug-02
interface mgmt
    ip static 10.99.99.1/30

! xyz-sw-ug-03
interface mgmt
    ip static 10.99.99.2/30
```

Ein `no ip dhcp` gibt es auf diesem Interface nicht. Das Setzen einer statischen Adresse
deaktiviert DHCP von selbst.

Der Distribution-Switch läuft auf einem anderen Firmware-Zweig, in dem `no routing` gar kein
gültiger Interface-Befehl ist und Ports ohnehin Layer 2 sind:

```
! xyz-sw-ug-01 — normale LAG, nicht multi-chassis
interface lag 21
    no shutdown
    lacp mode active
    vlan trunk native 50
    vlan trunk allowed all
```

### VSX

Zuerst den zweiten 100-GbE-Link und das Keepalive verkabeln, dann über das Keepalive pingen,
bevor es weitergeht.

```
! beide Member
interface 1/1/17,1/1/18
    no routing
    lag 128
    no shutdown

! xyz-sw-ug-02
vsx
    system-mac 02:00:00:00:01:00
    inter-switch-link lag 128
    role primary
    keepalive peer 10.99.99.2 source 10.99.99.1 vrf mgmt

! xyz-sw-ug-03
vsx
    system-mac 02:00:00:00:01:00
    inter-switch-link lag 128
    role secondary
    keepalive peer 10.99.99.1 source 10.99.99.2 vrf mgmt
```

Vor dem `vsx`-Block über den Keepalive pingen:

```
! von xyz-sw-ug-02
ping 10.99.99.2 vrf mgmt
```

Erst wenn `show vsx status` sauber ist, und keinen Moment früher:

```
xyz-sw-ug-03(config)# spanning-tree priority 4
```

### Die Uplink-LAG

Link für Link, und LACP zwischen beiden Hälften prüfen.

```
! xyz-sw-ug-02
interface 1/1/16
    lag 21
    no shutdown

! xyz-sw-ug-01
interface 1/1/52
    lag 21
    no shutdown
```

`show lacp interfaces` auf beiden Seiten, dann dasselbe für `xyz-sw-ug-03` `1/1/16` und
`xyz-sw-ug-01` `1/1/51` wiederholen.

### Die Firewall-LAGs

Die Member wurden bei der Vorbereitung angelegt und abgeschaltet gelassen, die Firewall an den
Core zu bringen ist also ein Befehl pro Port.

```
! xyz-sw-ug-02 und xyz-sw-ug-03
interface 1/1/15
    no shutdown

interface 1/1/7
    no shutdown
```

`write memory` auf jedem Switch, den man angefasst hat, dann `checkpoint diff startup-config
running-config`, um zu bestätigen, dass nichts ungespeichert geblieben ist.

### Konfigurationssynchronisation

Nicht am Aufbautag, sondern später hinzugefügt, und erst nachdem geprüft war, dass beide
Mitglieder ohnehin übereinstimmen. Nur auf dem Primary.

```
configure terminal

! jedes VLAN
vlan 10
    vsx-sync
    exit
! für 20, 30, 40, 50, 60, 70, 80 wiederholen

! jedes Multi-Chassis-LAG, aber nicht lag 128 — der ISL ist keines
interface lag 11
    vsx-sync vlans
    exit
! für lag 12, 21, 31-35, 41-44, 51 wiederholen

! ganze Feature-Kategorien
vsx
    vsx-sync lldp snmp ssh stp-global time
    exit

end
write memory
```

Danach prüfen, ob es angekommen ist und ob die beiden Mitglieder übereinstimmen:

```
show vsx status              # Config Sync Status: In-Sync
show vsx config-consistency
```

## Ressourcen

- [Aruba VSX configuration guide, AOS-CX 10.13](https://arubanetworking.hpe.com/techdocs/AOS-CX/10.13/PDF/vsx.pdf)
- [Aruba 8360 basic and VSX configuration (random-it-blog.de)](https://random-it-blog.de/posts/aruba-8360-basic-and-vsx-configuration-part-2/)

---

*Dieser Artikel wurde mit Unterstützung von Claude (Anthropic) verfasst.*
