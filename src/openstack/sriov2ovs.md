release time :2019-04-19 16:50

Because sometimes it is necessary to change the functions of the computing nodes, convert the sriov computing nodes into ovs computing nodes in batches.

Arrange the manually modified commands one by one to form a script, and then use the ansible tool to run the following script in batches to convert the sriov computing node to the ovs computing node.

    # cat sriov2ovs.bash

    # stop physical network card sriov vf
    echo '0' > /sys/class/net/ens1f0/device/sriov_numvfs
    sed -i '/sriov_numvfs/d' /etc/rc.d/rc.local

    # stop and disable neutron-sriov-nic-agent service， because the neutron ovs service is to be enabled
    systemctl disable neutron-sriov-nic-agent
    systemctl stop neutron-sriov-nic-agent

    # Delete nova.conf sriov passthrough whitelist configuration
    sed -i '/passthrough_whitelist/d' /etc/nova/nova.conf
    systemctl restart openstack-nova-compute

    # configure physical network card
    # configure bond
    (cat <<HERE
    TYPE=Ethernet
    BOOTPROTO=none
    NAME=ens1f0
    DEVICE=ens1f0
    ONBOOT=yes
    MASTER=bond1
    SLAVE=yes

    HERE
    )> /etc/sysconfig/network-scripts/ifcfg-ens1f0

    (cat <<HERE
    TYPE=Ethernet
    BOOTPROTO=none
    NAME=ens1f1
    DEVICE=ens1f1
    ONBOOT=yes
    MASTER=bond1
    SLAVE=yes

    HERE
    )> /etc/sysconfig/network-scripts/ifcfg-ens1f1

    (cat <<HERE
    BOOTPROTO=none
    DEVICE=bond1
    ONBOOT=yes
    ## MTU=1500
    TYPE=Bond
    BONDING_OPTS="mode=active-backup miimon=100"

    HERE
    )> /etc/sysconfig/network-scripts/ifcfg-bond1

    # Modify the verification configuration of the source address of the data packet
    sed -i '/net\.ipv4\.conf\.all\.rp_filter/d' /etc/rc.d/rc.local
    sed -i '/net\.ipv4\.conf\.default\.rp_filter/d' /etc/rc.d/rc.local
    sed -i '$a\net.ipv4.conf.all.rp_filter=0' /etc/sysctl.conf
    sed -i '$a\net.ipv4.conf.default.rp_filter=0' /etc/sysctl.conf
    sysctl -p

    # instal neutron-openvswitch,etc services
    yum install -y openstack-neutron openstack-neutron-ml2 openstack-neutron-openvswitch openvswitch

    # config file openvswitch_agent.ini, configure the mapping from the physical network card to the bridge, and restart the network-related services
    # neutron.conf
    # /etc/neutron/plugins/ml2/openvswitch_agent.ini
    (cat <<HERE
    [DEFAULT]

    [agent]
    extensions = qos

    [ovs]
    integration_bridge = br-int
    bridge_mappings = physnet1:br-prv

    [securitygroup]
    enable_ipset = true
    enable_security_group = true
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

    HERE
    )> /etc/neutron/plugins/ml2/openvswitch_agent.ini

    systemctl enable  openvswitch.service
    systemctl restart  openvswitch.service
    systemctl status  openvswitch.service

    # Add the bridge and network card connection that previously configured the mapping of the physical network card to the bridge
    ovs-vsctl add-br br-prv
    ovs-vsctl add-port br-prv bond1

    systemctl enable neutron-openvswitch-agent.service
    systemctl restart neutron-openvswitch-agent.service