release time :2019-09-20 18:49


Stein, the 19th version of OpenStack, supports 5G and edge computing.
OpenStack Stein enhances bare metal and network management performance, and enables faster startup of Kubernetes clusters. It also provides network upgrade functions for edge computing and NFV use cases, and enhances resource management and tracking performance.

* **OS version: CentOS 7.7 ( CentOS Linux release 7.7.1908 )**
* **Openstack version: Stein**

# After nova-compute is deployed on the compute node, it gets stuck at startup
View the nova-compute.log log, report message queue error

    2019-09-20 13:38:32.411 68483 INFO os_vif [-] Loaded VIF plugins: ovs, linux_bridge, noop
    2019-09-20 13:38:32.878 68483 ERROR oslo.messaging._drivers.impl_rabbit [req-b955a570-83bc-4a14-966f-2eefe5a82579 - - - - -] Connection failed: [Errno 113] EHOSTUNREACH (retrying in 2.0 seconds): error: [Errno 113] EHOSTUNREACH
    2019-09-20 13:38:34.888 68483 ERROR oslo.messaging._drivers.impl_rabbit [req-b955a570-83bc-4a14-966f-2eefe5a82579 - - - - -] Connection failed: [Errno 113] EHOSTUNREACH (retrying in 4.0 seconds): error: [Errno 113] EHOSTUNREACH
    2019-09-20 13:38:38.899 68483 ERROR oslo.messaging._drivers.impl_rabbit [req-b955a570-83bc-4a14-966f-2eefe5a82579 - - - - -] Connection failed: [Errno 113] EHOSTUNREACH (retrying in 6.0 seconds): error: [Errno 113] EHOSTUNREACH
    2019-09-20 13:38:44.912 68483 ERROR oslo.messaging._drivers.impl_rabbit [req-b955a570-83bc-4a14-966f-2eefe5a82579 - - - - -] Connection failed: [Errno 113] EHOSTUNREACH (retrying in 8.0 seconds): error: [Errno 113] EHOSTUNREACH
    2019-09-20 13:38:52.931 68483 ERROR oslo.messaging._drivers.impl_rabbit [req-b955a570-83bc-4a14-966f-2eefe5a82579 - - - - -] Connection failed: [Errno 113] EHOSTUNREACH (retrying in 10.0 seconds): error: [Errno 113] EHOSTUNREACH

Check the nova configuration file, the rabbitmq configuration is correct, log in to the controller node, check the nova service log, and there is no message queue error. Compare the configuration of the controller node and the compute node rabbitmq, the same, the controller node does not report an error, and the compute node reports an error. The message queue service is deployed on the controller node. It may be caused by the firewall that the nova service of the compute node cannot access the mq service of the controller node. Sure enough, the firewall is not closed, and the problem is solved after closing it.

# After deploying nova-compute on the compute node, execute nova service-list. The service of the compute node is normal, but the nova log of the compute node reports an error, which is related to resources. It seems to be related to the placement service.

    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager [req-128f79a5-8b18-471b-9967-aae3cfde3043 - - - - -] Error updating resources for node compute.: ResourceProviderRetrievalFailed: Failed to get resource provider with UUID 092506db-b8fe-49d3-a962-9182e0025dea
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager Traceback (most recent call last):
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager   File "/usr/lib/python2.7/site-packages/nova/compute/manager.py", line 8148, in _update_available_resource_for_node
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager     startup=startup)
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager   File "/usr/lib/python2.7/site-packages/nova/compute/resource_tracker.py", line 744, in update_available_resource
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager     self._update_available_resource(context, resources, startup=startup)
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager   File "/usr/lib/python2.7/site-packages/oslo_concurrency/lockutils.py", line 328, in inner
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager     return f(*args, **kwargs)
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager   File "/usr/lib/python2.7/site-packages/nova/compute/resource_tracker.py", line 825, in _update_available_resource
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager     self._update(context, cn, startup=startup)
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager   File "/usr/lib/python2.7/site-packages/nova/compute/resource_tracker.py", line 1032, in _update
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager     self._update_to_placement(context, compute_node, startup)
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager   File "/usr/lib/python2.7/site-packages/retrying.py", line 68, in wrapped_f
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager     return Retrying(*dargs, **dkw).call(f, *args, **kw)
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager   File "/usr/lib/python2.7/site-packages/retrying.py", line 223, in call
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager     return attempt.get(self._wrap_exception)
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager   File "/usr/lib/python2.7/site-packages/retrying.py", line 261, in get
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager     six.reraise(self.value[0], self.value[1], self.value[2])
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager   File "/usr/lib/python2.7/site-packages/retrying.py", line 217, in call
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager     attempt = Attempt(fn(*args, **kwargs), attempt_number, False)
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager   File "/usr/lib/python2.7/site-packages/nova/compute/resource_tracker.py", line 958, in _update_to_placement
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager     context, compute_node.uuid, name=compute_node.hypervisor_hostname)
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager   File "/usr/lib/python2.7/site-packages/nova/scheduler/client/report.py", line 873, in get_provider_tree_and_ensure_root
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager     parent_provider_uuid=parent_provider_uuid)
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager   File "/usr/lib/python2.7/site-packages/nova/scheduler/client/report.py", line 655, in _ensure_resource_provider
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager     rps_to_refresh = self._get_providers_in_tree(context, uuid)
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager   File "/usr/lib/python2.7/site-packages/nova/scheduler/client/report.py", line 71, in wrapper
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager     return f(self, *a, **k)
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager   File "/usr/lib/python2.7/site-packages/nova/scheduler/client/report.py", line 522, in _get_providers_in_tree
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager     raise exception.ResourceProviderRetrievalFailed(uuid=uuid)
    2019-09-20 14:20:24.477 69378 ERROR nova.compute.manager ResourceProviderRetrievalFailed: Failed to get resource provider with UUID 092506db-b8fe-49d3-a962-9182e0025dea

Search the Internet for this problem, which is related to permissions

    # vim /etc/httpd/conf.d/00-placement-api.conf
    # add
    <Directory /usr/bin>
        <IfVersion >= 2.4>
            Require all granted
        </IfVersion>
        <IfVersion < 2.4>
            Order allow,deny
            Allow from all
        </IfVersion>
    </Directory>
    # su -s /bin/sh -c "placement-manage db sync" placement
    # systemctl restart httpd

After modification, the problem is solved

    # sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-iptables: No such file or directory

Add in /etc/sysctl.conf:

    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    # When executing sysctl -p, it appears:
    [root@localhost ~]# sysctl -p
    sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-ip6tables: No such file or directory
    sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-iptables: No such file or directory
    
    # Solution:

    [root@localhost ~]# modprobe br_netfilter
    [root@localhost ~]# sysctl -p
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1


The module fails after restarting. The following is the script to automatically load the module at startup

Create a new rc.sysinit file in /etc/

    cat /etc/rc.sysinit

    #!/bin/bash
    for file in /etc/sysconfig/modules/*.modules ; do
    [ -x $file ] && $file
    done

    # Create a new file in the /etc/sysconfig/modules/ directory as follows

    cat /etc/sysconfig/modules/br_netfilter.modules
    modprobe br_netfilter
    
    # add permissions


    chmod 755 br_netfilter.modules
    
    # The module is automatically loaded after restart

    [root@localhost ~]# lsmod |grep br_netfilter
    br_netfilter           22209  0
    bridge                136173  1 br_netfilter

# The openstack virtual machine instance is stuck at system boot and cannot start the operating system

show

    booting from hard disk...
    GRUB`

Whether it is the cirros image downloaded from the Internet, or the linux and windows images created by installing and uploading by yourself, it cannot be started. It has been stuck for a while, and then moved to a physical machine. Install linux directly on the bare metal, and then install openstack. Everything is normal, and the virtual machine instance Both start normally, (windows need to install virtio driver).

Going back to solve the problem that the openstack installed on the virtual machine on vmware cannot start the instance operating system, and confirm the solution direction, it is the problem of the virtual disk format and driver. Through the method of virsh edit XXXX, it can be seen that the virtual machine cannot be started Is using the virtio driver

change it to Then start the virtual machine with virsh start XXX, it can be started normally, but soon, the instance is automatically shut down in less than 1 minute, for example, no matter whether you modify virsh edit XXX or modify the virtual machine /etc/libvirt/qemu/instan-00000002.xml The machine definition file is automatically restored to the original configuration file after the instance is started on the openstack interface.

Finally, I found a way to directly modify the parameter properties of the image file and specify the properties of the hard disk and network card:

    # openstack image set  --property hw_disk_bus=ide  --property hw_vif_model=e1000 <image-id>


This command modifies the hard disk attribute to ide, and the network card attribute to e1000

Then use this modified image to generate a virtual machine instance, ok, the system can be booted normally, and the virtual hard disk can be recognized.






