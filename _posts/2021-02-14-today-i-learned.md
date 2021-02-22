---
layout: post
title:  "TIL 2021-02-14"
date:   2021-02-14 22:00:00 +0200
categories: til learned ethernet
---

The digest of "Today I learned" topics.

# QoS

QoS (Quality of Service) is configured via special PCP field in 802.1Q VLAN support Ethernet header (standard 802.1p), shifting EtherType to right.
TPID in Ethertype = 0x8100 for 802.1Q, other values being valid for other Ethernet frame types.

# Jumbo Frames

Jumbo Frame is non standard extension to Ethernet standard, yet used by various hardware equipment which require high bandwidth. As an example, GigE Vision cameras often require enabling Jumbo Frames.

Max. size of Jumbo frame is 9000 bytes. Typical Ethernet frame is 1500 bytes, which is 1538 on Level 1 (physical), the number is defined so to avoid collisions (most probably, backward compatibility).