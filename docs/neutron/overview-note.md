# TUN/TAP

TUN and TAP are virtual network kernel interfaces. Being network devices supported entirely in software, they differ from ordinary network devices which are backed up by hardware network adapters.


Design

TUN (namely network TUNnel) simulates a network layer device and it operates with layer 3 packets like IP packets. TAP (namely network tap) simulates a link layer device and it operates with layer 2 packets like Ethernet frames. TUN is used with routing, while TAP is used for creating a network bridge.

Packets sent by an operating system via a TUN/TAP device are delivered to a user-space program which attaches itself to the device. A user-space program may also pass packets into a TUN/TAP device. In this case the TUN/TAP device delivers (or "injects") these packets to the operating-system network stack thus emulating their reception from an external source.
