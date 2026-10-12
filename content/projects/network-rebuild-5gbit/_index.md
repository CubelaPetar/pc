---
title: "Network infrastructure rebuild for a 5 Gbit/s line"
description: "A customer's internet connection was upgraded from 1 to 5 Gbit/s, and the network behind it was rebuilt to carry it."
date: 2026-10-12
orderByWeight: true
draft: false
---

A customer's internet connection was upgraded from a symmetric 1 Gbit/s line to a symmetric 5 Gbit/s line. The existing network could not carry that, so I rebuilt it: a dedicated WAN switch, a Sophos firewall cluster, an Aruba VSX core and LACP bonding down to every server and storage node.

The customer and the ISP are not named. Hostnames, IP addresses and anything else identifying have been changed throughout.

The project is documented in seven parts. Parts still to come:

3. Bench setup of the firewall cluster, Sophos XGS 4300: base configuration, VLANs, HA and LAGs
4. Firewall migration on the day: the dual WAN setup, the MC-LAG, and the arrival of the ISP
5. LACP bonding of the PVE server and the bare metal Linux compute nodes
6. LACP bonding of the Qumulo storage cluster
7. Setting up an NDR system as a VM, and the port mirroring configuration on the switch core

## Published parts
