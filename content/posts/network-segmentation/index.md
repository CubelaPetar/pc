---
title: "Network Segmentation"
description: "What network segmentation is, why NIS2 and the BSI care about it, and which segments make sense for which network size."
date: 2026-10-09
lastmod: 2026-10-09
draft: false
categories:
  - network
tags: 
  - network
  - vlans
  - firewall
  - nis2 
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

## Introduction

At work I do a lot of firewall migrations and network restructurings for small and mid-sized companies, and in almost every one of them network segmentation comes up. Usually we suggest it; only a few customers ask for it themselves. Some end up with a clean VLAN layout and a tight rule set. For others we decide to postpone it until the new firewall is in place, which is sometimes the better choice. The one thing I learned is that there is no universal layout for every infrastructure. **Which separation makes sense depends heavily on the size and the context of the network.** A small office, a school with hundreds of iPads and a building automation network with thousands of controllers need very different things.

Network segmentation means splitting one network into several smaller ones and putting a firewall between them. Devices that belong together sit in the same segment. Everything that wants to cross from one segment to another has to pass the firewall, and the firewall only lets through what you explicitly allowed.

The reason is simple. Sooner or later something in your network gets compromised: a laptop with a malicious attachment, a printer with ten year old firmware, a camera nobody remembers buying. In a flat network that device can talk to everything else, including the domain controller, the NAS and the backup server. In a segmented network it can only reach what its segment is allowed to reach. Segmentation does not stop the first infection. It limits how far an attacker (or e.g. ransomware) can move afterwards, the so-called *lateral movement*, and it makes the blast radius smaller. A device should only be able to connect to what it actually needs. This is a key concept of zero-trust. 

A sentence from an article by the CMU Software Engineering Institute sums it up well: you cannot prevent a cyber breach, but you can isolate one.

There are side benefits too. Smaller broadcast domains, easier troubleshooting because you know what lives where, and a firewall log that suddenly tells you who talks to whom.

In my opinion a properly segmented network, together with a proper firewall rule set, is one of the biggest security features any infrastructure can have. This article is about the concept: why regulations more or less require it, zones and segments, typical segments, and the pitfalls I ran into in practice. Customer names and anything identifying are left out. 

I live and work in Germany, so when it comes to rules and recommendations I look mostly at what the EU and the German authorities publish. The most important one for me is the BSI (*Bundesamt für Sicherheit in der Informationstechnik*), the German federal office for information security. Other countries have their own laws and agencies, but the network concepts are the same everywhere.

## Regulations: why segmentation is more or less required

In Germany and the EU, more and more companies have legal obligations for their IT security. I am still learning this area myself, so this is only a short overview of the German and EU side and not legal advice.

What I learned so far: the laws rarely say "you must have a VLAN for printers". They ask for appropriate measures according to the *state of the art* (*Stand der Technik*). What that means for a network is defined by standards like the BSI IT-Grundschutz or ISO 27001, and all of them ask for segmentation. Try to explain to an auditor how you limit the impact of an incident in a flat network.

| What | Type | Who has to comply | Mentions segmentation? |
|---|---|---|---|
| **NIS2** / German **NIS2UmsuCG** (new BSIG) | Law (EU directive, German implementation) | Companies in the listed sectors above a size threshold | Not by name in the law itself, but it is one of the obvious measures |
| **Implementing Regulation (EU) 2024/2690** | Law (EU regulation) | Certain digital providers under NIS2 (DNS, cloud, data centres, MSPs, …) | **Yes**, explicitly |
| **DORA** (EU 2022/2554) | Law (EU regulation) | Banks, insurers and other financial entities | **Yes**, in the technical standards that go with it |
| **GDPR**, Art. 32 | Law (EU regulation) | Everyone processing personal data | No, only "appropriate technical and organisational measures" |
| **Cyber Resilience Act** (EU 2024/2847) | Law (EU regulation) | Manufacturers of products with digital elements | No; it is about the products, not your network |
| **BSI IT-Grundschutz** (NET.1.1) | Standard | Mandatory for the federal administration; voluntary for everyone else | **Yes**, in detail |
| **ISO/IEC 27001:2022** | Standard | Voluntary, but often demanded by customers and contracts | Yes, control A.8.22 "Segregation of networks" |
| **IEC 62443** | Standard | Voluntary; the reference for OT and industrial networks | Yes, the whole "zones and conduits" model |
| **PCI DSS** | Industry standard | Contractually required for anyone processing card payments | Yes, and segmentation has to be tested |

NIS2 is the one everyone talks about. In Germany it has been in force since December 2025 through the NIS2UmsuCG, and it affects tens of thousands of companies that never had such obligations before. Companies have to check themselves whether they are in scope; nobody sends you a letter.

### BSI IT-Grundschutz NET.1.1

The document I like most is the BSI IT-Grundschutz module **NET.1.1 "Netzarchitektur und -design"** (Edition 2023). It is free, only ten pages, and very concrete. I am trying to follow it more and more in my projects. Its requirements have three levels:

- **Basis (B)**: *MUSS*. The bare minimum.
- **Standard (S)**: *SOLLTE*. Together with Basis this is what the BSI considers state of the art. You can deviate, but you have to justify it.
- **Erhöhter Schutzbedarf (H)**: suggestions for systems with high protection needs, decided by risk analysis.

The requirements I find most relevant for segmentation:

| ID | Level | Content (paraphrased) |
|---|---|---|
| A4 | B | At least three physically separated zones: internal network, DMZ, external connections. Firewalls between them that only forward explicitly allowed traffic. |
| A5 | B | Clients and servers in different segments, with at least a stateful packet filter in between. Guests and devices you don't control get their own segments. |
| A6 | B | Only devices with a similar security level in the same segment. |
| A9 | B | Decide for every network how trustworthy it is. Untrusted networks are treated like the internet. |
| A19 | S | Infrastructure services (DNS, DHCP, directory, …) in a dedicated segment. |
| A21 | S | Out-of-band management in its own segments. Management traffic must not be a way around the segmentation. |
| A22 | S | A written segmentation concept, including dev/test systems and network access control for client segments. |
| A23 | S | Different protection needs mean different segments. If they are mixed, the highest protection need counts for the whole segment. |
| A24 | S | VLANs must not be "overcome" (VLAN hopping). |
| A33 | H | Micro-segmentation. |
| A36 | H | No VLANs at all for *very* high protection needs. |

## Zones and segments

### Terms

People (me included) use "network", "VLAN", "zone" and "segment" more or less interchangeably. NET.1.1 is stricter about it, and it helps to know the difference:

- A **zone** is a large area with a clear trust level: the internal network, the DMZ, and the external connections (internet and other untrusted networks). According to the BSI, zones should always be **physically** separated and the transitions between them protected by a firewall.
- A **segment** is a subdivision *inside* a zone, e.g. clients, servers and printers inside the internal network. Segments may be logical, which in practice means VLANs.

In general, zones can also be other logical groupings of networks, like TRUSTED, UNTRUSTED, WIFI and so on. The important part is that every network belongs to exactly one zone, and the zone tells you how much you trust it.

### Start with the zones

Before I think about VLANs, I think about zones. They are the coarse structure, and the segments are filled in afterwards. The model I use is based on the BSI (NET.1.1 and the older ISi-LANA study) and looks like this:

| Zone | Trust | What lives there |
|---|---|---|
| External | untrusted | the internet, the ISP, partner and customer networks |
| DMZ | low | servers reachable from the internet: reverse proxy, mail relay, VPN gateway |
| Internal | trusted | the segments for servers, clients, printers, phones, … |
| Untrusted internal | untrusted | guest Wi-Fi, IoT, devices you don't control |
| Management | most sensitive | management interfaces of network devices and hypervisors, monitoring, logging, backup |

![The five zones and their segments, all connected through the firewall](zones.drawio.png)

A few rules follow from that:

- **Traffic is initiated from more trust towards less trust, not the other way round.** Clients may open connections into the DMZ. A server in the DMZ never opens a connection into the internal network. The BSI is explicit here: systems must not access the internal network from the internet or the DMZ.
- **Untrusted networks are treated like the internet.** This is BSI requirement A9, and I find it the most useful one. A guest Wi-Fi is physically inside your building, but from a security point of view it is the internet. It gets internet access and nothing else.
- **Management is the most sensitive zone.** Whoever reaches the management interfaces of the switches, the firewall and the hypervisors owns the network. More on that in the communication matrix.

Many firewalls know zones as a concept and let you write rules between zones instead of between single interfaces (on OPNsense, interface groups do a similar job). Even if yours doesn't, writing the zones down keeps the rule set consistent.

One honest remark: the BSI wants at least three *physically* separated zones and a two-tier firewall between the internet and the internal network. In the small and mid-sized companies I work with, I almost never see that. Usually one firewall (or one HA cluster) handles all zones, and the separation between them is logical. That is a compromise, and it's good to know that it is one.

### VLANs separate, firewall rules protect

Ten VLANs that are all routed to each other with an "allow any" rule are not segmentation. They are a flat network with extra steps. The security comes from the rule set between the segments, and that rule set should start with *deny everything* and then open what is needed. This is true for IPv4 and IPv6 alike. If you run both, every rule has to exist for both, otherwise the IPv6 side is wide open next to a carefully built IPv4 rule set.

## Typical segments

This is the layout I start with. The VLAN IDs are just a convention, but using the same IDs everywhere helps a lot when you look after many networks. In my day-to-day work MGMT is VLAN 50 at almost every customer, and I don't have to think about it.

**The common ones**

| Segment | Zone | What goes in | Why |
|---|---|---|---|
| MGMT | Management | Switch and AP management, firewall, hypervisor management interfaces, iLO/iDRAC/IPMI | Whoever controls these controls the network. Sometimes without internet access at all. |
| SERVER | Internal | Bare metal servers, VMs, NAS | Clients and servers separated (BSI A5) |
| CLIENT | Internal | Laptops, desktops, company smartphones | Where users and therefore most infections are |
| GUEST | Untrusted internal | Guest Wi-Fi, devices you don't control | Internet only, nothing internal |
| DMZ | DMZ | Servers reachable from the internet | If one is compromised, it must not lead into the internal network |

**Depending on size and context**

| Segment | Zone | What goes in | Why |
|---|---|---|---|
| INFRA | Internal | DNS, DHCP, domain controllers, NTP, RADIUS | Everyone needs them, so they need extra protection (BSI A19). They stay in the internal zone because every client has to reach them. |
| BACKUP | Management (or its own zone) | Backup server, backup storage | Ransomware goes for the backups first. Like management, it connects out to everything and nothing connects in. |
| MONIT / LOG | Management | Monitoring, syslog, SIEM | Part of the management area according to the BSI |
| IOT | Untrusted internal | Cameras, smart TVs, voice assistants, building sensors | Devices you can't patch or trust. The BSI's word for this would be a cage. |
| PRT | Internal | Printers | Old firmware, rarely updated, but they need to talk to many clients |
| VOICE | Internal | IP phones, PBX | QoS, and phones often come with their own provider |
| DEV / TEST | Internal | Lab and staging systems | NIS2 and BSI both ask for separation from production |
| SEC | Management | Vulnerability scanners and similar security tools | They need to reach everything, so nothing should reach them |
| VPN | Internal | Remote access users | Treat them like clients, or stricter |

A few things I want to point out:

- **The hypervisor appears twice.** Its management interface (the Proxmox or vCenter UI, the iLO) belongs to MGMT, its VMs to SERVER or wherever they fit. Otherwise the hypervisor becomes the bridge between your segments, exactly what BSI A21 warns about.
- **DMZ is not GUEST.** The DMZ is for *servers* you expose to the internet. Untrusted *clients* go to GUEST or IOT. Nobody plugs a laptop into the DMZ.
- **You don't need all of them.** In general every group of devices could get its own network, but every additional segment means more rules, more DHCP scopes and more things that break. A handful of segments with a strict rule set is better than twenty with sloppy ones.

## The communication matrix

The table of segments is only half of the concept. The other half is the question of who may talk to whom. I write this down as a matrix: rows are the source, columns the destination, and every cell that is not explicitly allowed is denied.

| from \ to | MGMT | INFRA | SERVER | CLIENT | BACKUP | PRT | IOT | GUEST | DMZ | Internet |
|---|---|---|---|---|---|---|---|---|---|---|
| **MGMT** | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | – | ✔ | updates only |
| **INFRA** | – | ✔ | – | – | – | – | – | – | – | DNS/NTP upstream |
| **SERVER** | – | ✔ | ✔ | – | – | print ports | – | – | – | restricted |
| **CLIENT** | – | DNS, DHCP, AD | app ports | – | – | print ports | specific | – | specific | ✔ |
| **BACKUP** | – | ✔ | pull | – | ✔ | – | – | – | – | – |
| **PRT** | – | DNS, DHCP | – | – | – | – | – | – | – | – |
| **IOT** | – | DNS | – | – | – | – | – | – | – | vendor cloud only |
| **GUEST** | – | – | – | – | – | – | – | – | – | ✔ |
| **DMZ** | – | – | specific | – | – | – | – | – | – | ✔ |

A few rules that are behind this matrix:

**Management connects out, nothing connects in.** MGMT may reach every segment to administer it. Nothing initiates a connection into MGMT, except an admin workstation or jump host that is itself in a protected segment. The BSI's management requirements go even further and split the management area into several segments of its own, which is a bit much for a small office.

**The backup server pulls.** Production systems should not be able to write to or delete from the backup storage. The backup server connects to the servers and pulls the data. If ransomware encrypts the file server, it can't reach the backups from there.

**Clients don't talk to clients.** In most offices there is no reason for two laptops to talk to each other directly. Blocking it (on the firewall it's the same segment, so this needs client isolation, see below) removes one of the most common ways ransomware spreads.

**Egress matters too.** The matrix is not only about inbound traffic. IoT devices only need their vendor's cloud, printers don't need the internet at all, and DNS from clients should go to your own resolver, not to whatever is hardcoded in the device.

A real example: we run a vulnerability scanner as a small VM in a dedicated SEC segment at several customers. Its rules are short: SEC to all internal segments allows ping and the services the scanner needs, SEC to the internet allows only HTTPS for updates, and everything towards SEC is denied. That is about as clean as a matrix row gets.

One practical tip from a bigger segmentation project: **build the rules on aliases first.** Before we moved a single device, we created aliases for "clients", "printers", "file server" and so on, and wrote all the new rules against them while everything was still in the flat network. When the devices moved to their new VLANs, we only had to change what the aliases contained, not the rules.

## Practical pitfalls

### Broadcast and multicast don't cross segments

This is the one that hits every segmentation project sooner or later. A lot of protocols only work inside one broadcast domain, and a router or firewall does not forward them:

- **mDNS / Bonjour** (UDP 5353). This is how a MacBook finds printers, AirPlay screens and file shares.
- **SSDP / UPnP**, used by Chromecast, Sonos, smart TVs.
- **NetBIOS and LLMNR** in older Windows networks.
- **DHCP**, which is also broadcast based.

DHCP is easy: the firewall serves DHCP on each VLAN interface, or you configure a DHCP relay.

mDNS is the hard one. As soon as printers and MacBooks are in different VLANs, the MacBooks won't find the printers automatically anymore. There are three ways I have dealt with this:

1. **An mDNS reflector or repeater.** At a school, teachers and students needed to find around 17 Apple TVs that lived in their own VLAN. We set up a small Debian VM running Avahi as reflector, with one interface in every VLAN involved, and opened UDP 5353 plus the AirPlay ports on the firewall. OPNsense also has an mDNS repeater plugin, which I used at another customer for printers. It works, but be aware what it does: it reflects *all* announced services between the selected VLANs. It punches a hole into your segmentation, so keep the firewall rules behind it tight.
2. **Keep the devices together.** For a small Mac-only office, I planned printers in the same VLAN as the clients for exactly this reason. Not ideal from a security point of view, but it is a conscious trade-off, and with a handful of printers it is easy to explain.
3. **Don't rely on discovery at all.** Deploy printers through a print server or the MDM, or publish the services via unicast DNS-SD records in your internal DNS. MacBooks can find printers in other subnets that way without any multicast.

### Inside a segment there is no firewall

Devices in the same VLAN talk to each other directly over the switch. The firewall never sees that traffic. So a segment only protects you from *other* segments. Some ways to reduce the risk inside one:

- client isolation on Wi-Fi, at least for guest and IoT networks
- private VLANs or protected ports on the switch
- host firewalls on the clients
- micro-segmentation for VMs, if your hypervisor supports a distributed firewall

The BSI requirement "only devices with a similar security level in the same segment" (A6) is basically the reason this matters.

### VLANs are not a security boundary by default

VLANs can be bypassed if the switches are configured carelessly (VLAN hopping). The basics:

- don't use VLAN 1 for anything; move the switch management off it too
- set the native VLAN on trunks to an unused dummy VLAN
- disable auto-trunking (DTP on Cisco), set access ports hard to access mode
- only allow the VLANs on a trunk that are needed there
- DHCP snooping, dynamic ARP inspection, and RA Guard where IPv6 is in use
- disable unused ports or put them in a parking VLAN

## How a segmentation project runs

Designing VLANs is the fun part. Moving devices is where the time goes. The biggest segmentation project I did was an office building with four floors, around ten rooms per floor, a switch per floor and almost each room had a printer. The target was MGMT, SERVER, CLIENT and PRINTER networks. What we did, and what I would do again:

- **Keep the old flat network as the server VLAN.** We simply renamed it. Servers kept their IP addresses, which avoids a lot of trouble with hardcoded IPs, DNS entries and licences bound to addresses. I have used this trick at almost every customer since.
- **Clients are easy.** They get their address by DHCP, so you change the VLAN on the switch port and they come back in the new network.
- **Printers are the hardest part.** Each printer got a new IP, a new hostname and had to be updated (or recreated) on the print server. We gave them a naming scheme with floor and number in the hostname and the same digits in the last octet of the IP (`…-og3-04` gets `x.x.70.34`), so you can tell the address from the label.
- **Label every wall socket.** We labelled all of them: the first port in every room `PRT`, the others `CLT`, and `MGMT` where an access point or a managed switch hangs. It sounds boring, but six months later nobody has to guess. In hindsight, I would also have put the switch or patch panel port on each socket label.
- **Go in phases.** First a pilot with a few clients and one printer, then all clients, then the printers floor by floor.

And sometimes the right decision is to not do it (yet). One customer wanted the new firewall first and postponed the segmentation. At another one we planned to move an old network into a VLAN and dropped the idea because the effort was bigger than the benefit. That's fine, as long as it is a conscious decision and written down somewhere.

## Sizing and process

### What fits which network

| Environment | Reasonable starting set |
|---|---|
| Small office | MGMT, SERVER, CLIENT, GUEST, maybe PRT and VOICE |
| Mid-sized company, possibly NIS2 | the above plus INFRA, DMZ, BACKUP, DEV/TEST, MONIT, VPN; a documented matrix |
| Large company / KRITIS | the above plus micro-segmentation, physically separated management, encryption on the wire |

### Segmentation is a process

1. **Inventory.** You can't segment what you don't know. At one customer the "core switch" turned out to be an unmanaged switch in the kitchen, and nobody knew how the cabling ran. That project starts with discovery, not with VLANs. The SEI article has a good point here: aim for *good enough* information, not perfect information, or you never get started. Also check what you already have. Most managed switches and firewalls can do VLANs today, the feature just isn't used.
2. **Classify** devices by protection need and trust.
3. **Define** zones and segments. Start with the most critical systems and give them their own segment first, instead of trying to redesign everything at once.
4. **Write the communication matrix**, default deny. Look at what traffic your critical systems actually need, and only allow that.
5. **Implement**: VLANs on firewall, switches and access points, rules, DHCP.
6. **Test.** From each segment, try to reach every other segment (`nmap` is enough) and check that only what the matrix allows works.
7. **Log** denied traffic between segments and actually look at it. It shows misconfigurations, and sometimes things that should not be there at all.
8. **Review** regularly, at least once a year and after every bigger change. Both the BSI and NIS2 ask for this.

The step people forget is the one after the project: every new server, printer or access point has to land in the right segment, and every new application needs its flows added to the matrix. If that is not part of the normal change process, the network slowly becomes flat again. And as boring as it sounds, it needs backing from management. Segmentation changes how people work (the printer is suddenly gone, the NAS is not found anymore), and without that backing the first complaint ends with an "allow any" rule.

## Conclusion

Technically, network segmentation is a few VLANs, a firewall in between and a rule set that starts with "deny everything". The configuration takes an afternoon. The planning, the documentation and moving the devices take weeks.

My advice: start with the zones, then the segments. Separate management, servers, clients and guests. Write down who may talk to whom. Test it. Look at the logs. Extend it when you know where the real risks are, instead of designing twenty segments on the first day. And if you are looking for a reference, read BSI NET.1.1. It's only ten pages.

And one last thing, because "zero-trust" comes up in every vendor presentation: I see segmentation as a prerequisite for zero-trust rather than something it replaces. You need to know your segments and flows before you can start making decisions per user and per device.

## Sources

**Laws and official documents**

- [NIS2 Directive (EU) 2022/2555](https://eur-lex.europa.eu/eli/dir/2022/2555/oj)
- [Implementing Regulation (EU) 2024/2690](https://eur-lex.europa.eu/eli/reg_impl/2024/2690/oj)
- [BSI IT-Grundschutz NET.1.1 Netzarchitektur und -design (Edition 2023)](https://www.bsi.bund.de/SharedDocs/Downloads/DE/BSI/Grundschutz/IT-GS-Kompendium_Einzel_PDFs_2023/09_NET_Netze_und_Kommunikation/NET_1_1_Netzarchitektur_und_design_Edition_2023.pdf?__blob=publicationFile&v=3)
- [BSI ISi-L: Secure Connection of Local Networks to the Internet (2008)](https://www.bsi.bund.de/SharedDocs/Downloads/EN/BSI/InternetSecurity/ISi-LANA-ISi-L.pdf?__blob=publicationFile&v=1), old, but the principles still hold
- [NIST SP 800-207 Zero Trust Architecture](https://csrc.nist.gov/pubs/sp/800/207/final)

**Articles**

- [CMU SEI: Network Segmentation: Concepts and Practices (2020)](https://www.sei.cmu.edu/blog/network-segmentation-concepts-and-practices/)
- [it-planet: NIS2 and network segmentation, a practical guide](https://blog.it-planet.com/en/nis2-and-network-segmentation-a-practical-guide/)
