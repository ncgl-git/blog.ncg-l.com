---
title: "Linux: Bridging 2 Bonded Ethernet Ports"
date: 2025-04-28T20:19:38-04:00
draft: false
categories: [developer guide]
---

Simple /etc/network/interfaces file to show how to bond 2 ethernet ports together, and add that bond to a bridge:

```declarative
auto enp9s0
iface enp9s0 inet manual
        bond-master bond0

auto enp10s0
iface enp10s0 inet manual
        bond-master bond0

auto bond0
iface bond0 inet manual
        bond-slaves enp9s0 enp10s0
        bond-mode 802.3ad
        bond-miimon 100
        bond-downdelay 200
        bond-updelay 200

auto vmbr0
iface vvmbr0 inet static
        address 192.168.2.100/24
        gateway 192.168.2.1
        bridge-ports bond0
        bridge-stp on
        bridge-fd 2
```

Then run:
```declarative
>>> ifdown enp9s0
>>> ifdown enp10s0
>>> ifup bond0
>>> ifup vmbr0
```