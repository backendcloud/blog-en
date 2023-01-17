release time :2017-06-22 09:16

# nova evacuate

Various implementation schemes basically call nova's evacuate interface without exception.

nova evacuate is very similar to hot migration. They all want to transfer instances from one node to another. The difference is mainly that thermal migration is performed under normal conditions, while evacuation is performed under abnormal conditions. A vivid example is that thermal migration occurs before an earthquake to escape from a building, and evacuation occurs after an earthquake occurs by escaping from a destroyed building.

But strictly speaking, the above analogy is not rigorous enough. To be precise, it is not escaped from the destroyed building, but resurrected.

So the translation of nova evacuate into resurrection is more accurate than evacuation.

# masakari

NTT NTT's open source project masakari has become an independent project of Openstack, which is dedicated to compute node ha.

The Japanese word for masakari is まさかり, meaning an axe. To be precise, it's an axe. It is the ax used by Zhou Xingchi's movie Ax Gang. It is not known whether the initiators of the project, several Japanese, were affected by this.

The name masakari is to find the location of the fault, use the ax to help the highest stunt, and throw the ax accurately to isolate and cut off the fault.

# detect 3 type of vm down
* unexpected vm down (monitoring libvirt's events)
* vm manager down (monitoring manager process)
* host down (using pacemaker)

# New features that masakari is working on

1) Now it only supports evacuation of vm in active, stopped, and error states.
Later, it can be added to evacuate vm in more states, such as shelved, rescued, paused and suspended

2) Now the process after the fault is detected is fixed, and the only way to change it is to change the code.
In the future, users can customize the process by writing yaml files.

# compute node ha other technologies used
* consul
* raft
* gossip

# compute node ha other related open source projects
* Openstack Congress (Policy as a Service)
* pacemaker-remote (host monitor solution)
* mistral-evacuate (workflow)

# Appendix 1: Gossip protocol

A Gossip protocol runs between all Consul Agents (both server and normal). Both server nodes and ordinary Agents will join this Gossip cluster to send and receive Gossip messages. Every once in a while, each node will randomly select several nodes to send Gossip messages, and other nodes will randomly select other nodes to relay and send messages again. After such a period of time, the entire cluster can receive this message.

At first glance, it seems that the efficiency of this sending method is very low, but there have been papers in mathematics that have demonstrated its feasibility, and the Gossip protocol is already a relatively mature protocol in the P2P network. You can check the introduction of Gossip, which contains a simulator that can tell you the time and bandwidth required for messages to propagate in the cluster. The biggest advantage of the Gossip protocol is that even if the number of cluster nodes increases, the load on each node will not increase much and is almost constant. This allows Consul-managed clusters to scale out to thousands of nodes.

# Appendix 2: Path Algorithms

Find the optimal path for evacuate.

Not only for compute node HA, but also for load optimization and balancing

Try to design the migration path algorithm to optimize the performance of the node where the VM is located to maximize the return on hardware investment.