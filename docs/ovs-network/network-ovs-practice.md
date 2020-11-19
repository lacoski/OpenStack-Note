[root@com01 ~]# virsh dumpxml instance-00000006
    <interface type='bridge'>
      <mac address='fa:16:3e:86:08:69'/>
      <source bridge='qbr2b697681-2d'/>
      <target dev='tap2b697681-2d'/>
      <model type='virtio'/>
      <driver name='qemu'/>
      <mtu size='1450'/>
      <alias name='net0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>


[root@com01 ~]# brctl show
bridge name	        bridge id		    STP enabled	    interfaces
qbr2b697681-2d		8000.7a20c9081eb9	    no		    qvb2b697681-2d
							                            tap2b697681-2d

[root@com01 ~]# ovs-vsctl show
    Bridge br-int
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        datapath_type: system
        Port "int-br-eth1"
            Interface "int-br-eth1"
                type: patch
                options: {peer="phy-br-eth1"}
        Port "qvo2b697681-2d"
            tag: 2
            Interface "qvo2b697681-2d"
        Port br-int
            Interface br-int
                type: internal
        Port "tapaeb67409-5f"
            tag: 1
            Interface "tapaeb67409-5f"
                type: internal
        Port "tap35ff9ac4-7d"
            tag: 2
            Interface "tap35ff9ac4-7d"
                type: internal
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}

[root@com01 ~]# ip netns
qdhcp-7ab317fc-9aae-451e-846d-4042cea894ee (id: 1)
qdhcp-0b57be29-87e4-41b8-bc23-4568e805fdf7 (id: 0)
