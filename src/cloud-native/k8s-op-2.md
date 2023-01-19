release time :2022-11-17 18:56

# The CentOS Stream 8 Pod network cannot communicate across nodes
Environmental information:
* OS: CentOS Stream 8
* K8S CNI: calico
The same deployment works fine on CentOS 7, but once switched to CentOS Stream 8, the network becomes abnormal. The specific performance is that node->other node pod, pod->other node cannot communicate, but the node can communicate with the pod.

solution:

Reference: https://github.com/projectcalico/calico/issues/3709

iptables-legacy and iptables-nft

iptables-legacy and iptables-nft are two versions of the iptables command. The two cannot be used at the same time, you can only choose one of the two, centos7 defaults to iptables-legacy, centos8 defaults to iptables-nft, iptables-nft and iptables-legacy are different things, iptables-nft is an upgraded version of iptables-legacy. Although the commands are exactly the same, iptables-nft is 100% compatible with the command format of iptables-legacy.

> The subtitle should be iptables and iptables-nft to be precise, as of iptables 1.8, the iptables command-line client comes in two different versions/modes: "legacy", which uses the kernel iptables API, just like iptables 1.6 and earlier , "nft", which translates the iptables command-line API to the kernel nftables API. To make a good distinction, let's use iptables-legacy and iptables-nft.

To check whether the operating system is using iptables-legacy or iptables-legacy. Two ways:

One is the iptables -V command. Generally, the one with nf_tables is iptables-nft, and the one without iptables-legacy.

```bash
[root@k8s-11 ~]# iptables -V
iptables v1.4.21
```

```bash
root@rhel-8 # iptables -V
iptables v1.8.4 (nf_tables)
```

The other is ls -al /usr/sbin/iptables command, with nft is iptables-nft, without iptables-legacy.

```bash
[root@k8s-11 ~]# ls -al /usr/sbin/iptables
lrwxrwxrwx. 1 root root 13 Jan 21  2022 /usr/sbin/iptables -> xtables-multi
```

```bash
root@rhel-8 # ls -al /usr/sbin/iptables
lrwxrwxrwx. 1 root root 17 Mar 17 10:22 /usr/sbin/iptables -> xtables-nft-multi
```

Add environment variables in calico-node to solve the problem:

    env:
    - name: FELIX_IPTABLESBACKEND
    value: NFT


In the RHEL 8 series, the iptables version is 1.8.2, which is based on nftables. And the RHEL 8 series does not provide a way to switch to legacy mode.

If the CNI plug-in (calico) uses iptables, the above environment variables need to be added.

> The default environment variable is auto, but the detection mechanism of auto is not perfect, and there will be problems. I don't know if the latest version has changed.

Attachment: general analysis ideas for calico network troubleshooting

If the cross-node Calico Pod network is unreachable: Pod A (the host is Host A) to Pod B (the host is Host B) the network is unreachable, and the common reasons for the network incompatibility are:
* Routing within a Pod is lost
* Host route lost
* iptables rule problem
* IPVS rule issue
* IP conflict
* Pod NIC stops working
* ARP table error
* Core DNS resolution problem
* traffic forwarding table problem


# Why does all pods have a route to 169.254.1.1 when Kubernetes CNI uses calico?
In a Calico network, each host acts as a gateway router for the workload it hosts. In a container deployment, Calico uses 169.254.1.1 as the address of the Calico router. By using link-local addresses, Calico saves valuable IP addresses and relieves the user of the burden of configuring appropriate addresses. While routing tables may seem a little strange to someone who is used to configuring LAN networks, it is fairly common to use explicit routing in WAN networks without subnet local gateways.

Calico assigns the interface with the same name for all pods: cali0, and assigns the same mac address on calico0: ee:ee:ee:ee:ee:ee, because calico only cares about the IP address of the third layer, and does not care about the MAC address of the second layer at all. . And assigned the same default route default via 169.254.1.1 dev cali0

All packets will be sent to the next hop 169.254.1.1 through cali0 (this is the reserved local IP network segment). This is the choice made by calico to simplify the network configuration. The routing rules in the container are the same, no need dynamic updates.

cali0 is one end of the veth pair, and its opposite end is the interface named by caliXXXX on the host. You can use ethtool -S cali0 to list the interface idnex of the opposite end.

Then on the host machine, you can find the network card information whose interface is idnex, and check the network card information of the host machine. This interface has a randomly assigned MAC address, but no IP address. After the interface receives the ARP request, it directly responds, and the MAC address in the response message is the MAC address of the interface. In other words, it replies with its own MAC address to the container. The IP address of the subsequent packets of the container is still the destination container, but the MAC address becomes the address of the interface on the host, that is to say, all packets will be sent to the host, and then the host will forward it according to the IP address.

The interface of the host, regardless of the content of the ARP request, directly responds with its own MAC address, which is called an ARP proxy, which is enabled by calico, and can be confirmed by the following command:

    # cat /proc/sys/net/ipv4/conf/calif24874aae57/proxy_arp
    1

In general, it can be considered that calico uses the host as the default gateway of the container, and all packets are sent to the host, and then the host forwards it according to the routing table. Different from the classic network architecture, calico does not configure an IP address for the default network (so that each network will consume an additional IP resource, and the corresponding IP address and routing information will be added to the host), but through arp proxy and modify the container routing table to achieve.

After the interface of the host receives the message, the following things are easy to understand. All the messages will go according to the routing table of the host.

# cni path problem analysis leads to Pod creation error
An error was reported when creating a pod:

    remote_runtime.go:116] "RunPodSandbox from runtime service failed" err="rpc error: code = Unknown desc = failed to setup network for sandbox 

Cause Analysis:

The reason is that the container engine docker is switched to containerd, and the CNI bin paths of the two are inconsistent.

The parameter kubernetes/kubelet.env of kubelet is configured as –cni-bin-dir=/etc/cni/bin

calico-node deployment yaml:

    volumes:
    - hostPath:
        path: /etc/cni/bin
        type: ""
    name: cni-bin-dir
    - hostPath:
        path: /etc/cni/net.d
        type: ""
    name: cni-net-dir


The cni bin file of containerd is placed in /opt/cni/bin. Make a soft link to the directory or configure a new directory path.

# namespace or ingress cannot be deleted for a long time


You can delete the content in the finalizers in the yaml file corresponding to the resource, and you can delete it.

> This method is simple and violent. However, Kubernetes uses this field to design a deletion interceptor, and it needs to wait for the resources in this field to be cleared before it can be actually deleted. Therefore, it is best to analyze the specific reasons why it cannot be deleted, find out the reason, and delete it naturally.
