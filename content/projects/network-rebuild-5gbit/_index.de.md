---
title: "Neuaufbau einer Netzwerkinfrastruktur für eine 5-Gbit/s-Leitung"
description: "Der Internetanschluss eines Kunden wurde von 1 auf 5 Gbit/s aufgerüstet, und das Netzwerk dahinter wurde neu aufgebaut, um ihn zu tragen."
date: 2026-10-12
orderByWeight: true
draft: false
---

Der Internetanschluss eines Kunden wurde von einer symmetrischen 1-Gbit/s-Leitung auf eine symmetrische 5-Gbit/s-Leitung aufgerüstet. Das bestehende Netzwerk konnte das nicht tragen, also habe ich es neu aufgebaut: ein dedizierter WAN-Switch, ein Sophos-Firewall-Cluster, ein Aruba-VSX-Core und LACP-Bonding bis zu jedem Server und Storage-Node.

Kunde und ISP werden nicht genannt. Hostnamen, IP-Adressen und alles andere Identifizierende wurden durchgehend geändert.

Das Projekt ist in sieben Teilen dokumentiert. Noch ausstehende Teile:

3. Bench-Aufbau des Firewall-Clusters, Sophos XGS 4300: Grundkonfiguration, VLANs, HA und LAGs
4. Firewall-Migration am Tag selbst: das Dual-WAN-Setup, das MC-LAG und die Ankunft des ISP
5. LACP-Bonding des PVE-Servers und der Bare-Metal-Linux-Compute-Nodes
6. LACP-Bonding des Qumulo-Storage-Clusters
7. Aufbau eines NDR-Systems als VM und die Konfiguration des Port-Mirrorings auf dem Switch-Core

## Veröffentlichte Teile
