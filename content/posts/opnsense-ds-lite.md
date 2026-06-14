---
title: "OPNsense with DS-Lite"
description: "How to configure OPNsense to work with a DS-Lite connection, including PPPoE passthrough, GIF tunnel setup, and IPv6 prefix delegation."
date: 2026-06-05
draft: true
authors:
  - Petar Cubela
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

# Setup OPNsense firewall with DS-lite at home

## Introduction

I have a DS-Lite connection at my WAN at home. I'm using a OPNsense firewall and needed to combine the two. 
First I thought I should contact my ISP and ask if I could get a proper dual stack setup with public IPv4 and IPv6 addresses
especially because of problems with VPN access from networks which are IPv4 only. But I decided this would be great to take
the IPv6 adoption more serious and to try to deal with an IPv6-only connection at the WAN-side. 
I found a nice solution to access my home network via VPN from IPv4-only networks which will be presented in a later article.
I was not really sure if the OPNsense firewall will work with DS lite, but I managed to get it working which I will present here. 
The OPNsense is able to work with DS-lite. Though it was not without problems.

## What is DS lite?

Something very annoying. 

- dynamic public IPv6 address.
- CGNAT IPv4 address (AFTR / RFC6333).

## OPNsense Configuration 

- FritzBox in PPPoE-Passthrough mode (get a draytek modem. g.fast required).
- IPv6 via PPPoE on some VLAN (FritzBox does this automatically).
- IPv4 via a GIF tunnel (needs IPv6).
- Setup up PPPoE connection under devices.
- WAN interface config: 
    - IPv6 via DHCPv6. Client Conf: (Prefix Delegation Size: 56, Send Prefix Hint: True, Optional Prefix ID: 9).
    - IPv4: None.
- LAN uses 'Track Interface' for IPv6.


## Resources 

- [DS-Lite on 23.7.6+](https://forum.opnsense.org/index.php?topic=37813.0)
- <https://github.com/opnsense/core/issues/7713>
- <https://forum.opnsense.org/index.php?topic=46665.0>
- <https://forum.opnsense.org/index.php?topic=27935.msg136305#msg136305>
- <https://cybercyber.org/m-net-ds-lite-anschluss-mit-pfsense.html>
- <https://github.com/opnsense/core/issues/7713>

