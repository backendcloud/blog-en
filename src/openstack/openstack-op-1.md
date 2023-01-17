release time :2018-08-26 18:38

# Troubleshooting process

* First determine the resource ID of the fault, and determine the component where the fault occurred
* Check the log of the corresponding component, search according to the fault resource ID, and find the corresponding ERROR log
* If the ERROR log points the problem to other components, then according to the resource ID, time, req-id and other information in the ERROR log, other components continue to search for the problem until the root cause of the problem is found.
* If no ERROR is found in the log, it may be that the network is not connected, causing the request to fail to reach the API. At this time, you need to check the connectivity with the API (if you use VIP, you need to check the connectivity with the VIP and the connectivity of the real IP separately. ).
* If the corresponding request can be found in the API, but the conductor/scheduler/compute does not find the corresponding log, it may be that the MQ is faulty.
* If the component has not refreshed any logs for a long time, the component process may hang or be in a dead state. You can try to restart the service, or open Debug first and then restart the service. 

# create vm error

    [root@EXTENV-194-18-2-16 nova]# cat nova-compute.log | grep 620cd801-8849-481a-80e0-2980b6c8dba6
    2018-08-23 15:23:36.136 3558 INFO nova.compute.resource_tracker [req-f76d5408-00f8-4a67-854e-ad3da2098811 - - - - -] Instance 620cd801-8849-481a-80e0-2980b6c8dba6 has allocations against this compute host but is not found in the database.

Analysis: It feels that the information database of node is out of sync

nova show error vm, package cell error

    #### Each time a computing node is added, the control node needs to execute:
    # su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova

problem solved.

# The neutron service is good, and the command line creates a network to view the network normally, but the dashboard cannot view network-related pages

The dashboard network page reports an error Invalid service catalog service: network

Analysis: It should be that Keystone is not properly configured. As a result, no relevant Catalog information was found.


    [root@EXTENV-194-18-2-11 ~]# openstack catalog list
    +-----------+-----------+-----------------------------------------------+
    | Name      | Type      | Endpoints                                     |
    +-----------+-----------+-----------------------------------------------+
    | placement | placement | RegionOne                                     |
    |           |           |   internal: http://nova-ha-vip:8778           |
    |           |           | RegionOne                                     |
    |           |           |   admin: http://nova-ha-vip:8778              |
    |           |           | RegionOne                                     |
    |           |           |   public: http://nova-ha-vip:8778             |
    |           |           |                                               |
    | keystone  | identity  | RegionOne                                     |
    |           |           |   public: http://keystone-ha-vip:5000/v3/     |
    |           |           | RegionOne                                     |
    |           |           |   internal: http://keystone-ha-vip:35357/v3/  |
    |           |           | RegionOne                                     |
    |           |           |   admin: http://keystone-ha-vip:35357/v3/     |
    |           |           |                                               |
    | glance    | image     | RegionOne                                     |
    |           |           |   admin: http://glance-ha-vip:9292            |
    |           |           | RegionOne                                     |
    |           |           |   internal: http://glance-ha-vip:9292         |
    |           |           | RegionOne                                     |
    |           |           |   public: http://glance-ha-vip:9292           |
    |           |           |                                               |
    | nova      | compute   | RegionOne                                     |
    |           |           |   public: http://nova-ha-vip:8774/v2.1        |
    |           |           | RegionOne                                     |
    |           |           |   admin: http://nova-ha-vip:8774/v2.1         |
    |           |           | RegionOne                                     |
    |           |           |   internal: http://nova-ha-vip:8774/v2.1      |
    |           |           |                                               |
    | neutron   | network   |                                               |
    | neutron   | network   | RegionOne                                     |
    |           |           |   public: http://neutron-server-ha-vip:9696   |
    |           |           | RegionOne                                     |
    |           |           |   admin: http://neutron-server-ha-vip:9696    |
    |           |           | RegionOne                                     |
    |           |           |   internal: http://neutron-server-ha-vip:9696 |
    |           |           |                                               |
    +-----------+-----------+-----------------------------------------------+



So just delete the first piece of data without url, but found that there is only openstack catalog list, no openstack catalog delete command. Later, I checked the keystone configuration file keystone.conf and found the following configuration
see [catalog]

From the configuration file and some information, it can be seen that the catalog is the data read from mysql, and then the dirty data is found in the service table in the keystone library of mysql, and then I know to use openstack service delete to delete the 'dirty data', the problem is solved.

    MariaDB [keystone]> select * from service;
    +----------------------------------+-----------+---------+-------------------------------------------------------------+
    | id                               | type      | enabled | extra                                                       |
    +----------------------------------+-----------+---------+-------------------------------------------------------------+
    | 520f6bf8564240be9678c4ef25305cad | placement |       1 | {"description": "OpenStack Placement", "name": "placement"} |
    | 960580852a594c078e68fe3683e35db5 | identity  |       1 | {"name": "keystone"}                                        |
    | 98ed18fcd8104732919bb5869a5a6dc2 | image     |       1 | {"description": "OpenStack Image", "name": "glance"}        |
    | abef1b9469d94d3ab9f27c8ed72a5a48 | compute   |       1 | {"description": "OpenStack Compute", "name": "nova"}        |
    | e37085e8fb2a49c0921c2d24f5e4f9b5 | network   |       1 | {"description": "OpenStack Networking", "name": "neutron"}  |
    | f1b661407ce04f79bc24605fa59bb74c | network   |       1 | {"description": "OpenStack Networking", "name": "neutron"}  |
    +----------------------------------+-----------+---------+-------------------------------------------------------------+
    6 rows in set (0.00 sec)

    MariaDB [keystone]> select * from endpoint;
    +----------------------------------+--------------------+-----------+----------------------------------+-----------------------------------+-------+---------+-----------+
    | id                               | legacy_endpoint_id | interface | service_id                       | url                               | extra | enabled | region_id |
    +----------------------------------+--------------------+-----------+----------------------------------+-----------------------------------+-------+---------+-----------+
    | 142cb619cd2242828b0c9394d5baaea1 | NULL               | public    | f1b661407ce04f79bc24605fa59bb74c | http://neutron-server-ha-vip:9696 | {}    |       1 | RegionOne |
    | 2252d3ef840b4c5aa1184ebe8d6094f1 | NULL               | public    | abef1b9469d94d3ab9f27c8ed72a5a48 | http://nova-ha-vip:8774/v2.1      | {}    |       1 | RegionOne |
    | 476654c6e7dd4d22b290de451e3afda0 | NULL               | admin     | abef1b9469d94d3ab9f27c8ed72a5a48 | http://nova-ha-vip:8774/v2.1      | {}    |       1 | RegionOne |
    | 562a5d5443af4dfab6760204d0adf3bf | NULL               | internal  | 520f6bf8564240be9678c4ef25305cad | http://nova-ha-vip:8778           | {}    |       1 | RegionOne |
    | 58bd5f09811a4ebcb62a4b51fb7ae444 | NULL               | admin     | f1b661407ce04f79bc24605fa59bb74c | http://neutron-server-ha-vip:9696 | {}    |       1 | RegionOne |
    | 600811f8ccaf42669d4d83b897af3933 | NULL               | admin     | 520f6bf8564240be9678c4ef25305cad | http://nova-ha-vip:8778           | {}    |       1 | RegionOne |
    | 80683f619efb41dcbb6796ea04f16159 | NULL               | internal  | f1b661407ce04f79bc24605fa59bb74c | http://neutron-server-ha-vip:9696 | {}    |       1 | RegionOne |
    | 8e0a684607294a729f87d7d8b1a639ca | NULL               | public    | 520f6bf8564240be9678c4ef25305cad | http://nova-ha-vip:8778           | {}    |       1 | RegionOne |
    | 9ef0f18d891e45608ffc41985dc6afa6 | NULL               | public    | 960580852a594c078e68fe3683e35db5 | http://keystone-ha-vip:5000/v3/   | {}    |       1 | RegionOne |
    | a0b10cb04a5b4ca3859aaf2ea4ca2a3b | NULL               | admin     | 98ed18fcd8104732919bb5869a5a6dc2 | http://glance-ha-vip:9292         | {}    |       1 | RegionOne |
    | c53979becccc44f1813e9f50a619af7e | NULL               | internal  | 960580852a594c078e68fe3683e35db5 | http://keystone-ha-vip:35357/v3/  | {}    |       1 | RegionOne |
    | dadbb8dc218245bbba8c9a34237413ec | NULL               | internal  | 98ed18fcd8104732919bb5869a5a6dc2 | http://glance-ha-vip:9292         | {}    |       1 | RegionOne |
    | f4034b8c086a451caed52ac51a761fb0 | NULL               | public    | 98ed18fcd8104732919bb5869a5a6dc2 | http://glance-ha-vip:9292         | {}    |       1 | RegionOne |
    | fc150884825544baaf4912f14e76f51a | NULL               | internal  | abef1b9469d94d3ab9f27c8ed72a5a48 | http://nova-ha-vip:8774/v2.1      | {}    |       1 | RegionOne |
    | fc7132052063438895674fd7b840db68 | NULL               | admin     | 960580852a594c078e68fe3683e35db5 | http://keystone-ha-vip:35357/v3/  | {}    |       1 | RegionOne |
    +----------------------------------+--------------------+-----------+----------------------------------+-----------------------------------+-------+---------+-----------+
    15 rows in set (0.00 sec)

    [root@EXTENV-194-18-2-11 ~]#  openstack service list
    +----------------------------------+-----------+-----------+
    | ID                               | Name      | Type      |
    +----------------------------------+-----------+-----------+
    | 520f6bf8564240be9678c4ef25305cad | placement | placement |
    | 960580852a594c078e68fe3683e35db5 | keystone  | identity  |
    | 98ed18fcd8104732919bb5869a5a6dc2 | glance    | image     |
    | abef1b9469d94d3ab9f27c8ed72a5a48 | nova      | compute   |
    | e37085e8fb2a49c0921c2d24f5e4f9b5 | neutron   | network   |
    | f1b661407ce04f79bc24605fa59bb74c | neutron   | network   |
    +----------------------------------+-----------+-----------+
    [root@EXTENV-194-18-2-11 ~]#  openstack service delete e37085e8fb2a49c0921c2d24f5e4f9b5
    [root@EXTENV-194-18-2-11 ~]# systemctl restart httpd.service memcached.service




# When viewing the image named in Chinese, an error is reported

    [root@NFJD-TESTVM-CORE-API-1 ~]# glance image-list
    'ascii' codec can't encode character u'\u5982' in position 1242: ordinal not in range(128)

Analysis: The image name is named in Chinese.

Add export LC_ALL=zh_CN.UTF-8

to /etc/profile

At the same time, pay attention to whether there is export LC_ALL=C in the source file

# vm build failed

    2017-05-25 11:01:29.577 21880 TRACE nova.compute.manager [instance: 2c5a8e62-62d0-430d-8747-795350bb6939] ProcessExecutionError: Unexpected error while running command.
    2017-05-25 11:01:29.577 21880 TRACE nova.compute.manager [instance: 2c5a8e62-62d0-430d-8747-795350bb6939] Command: qemu-img convert -O raw /var/lib/nova/instances/_base/f5797db00aacfbe240bbfb0f53c2da80e4be6dfc.part /var/lib/nova/instances/_base/f5797db00aacfbe240bbfb0f53c2da80e4be6dfc.converted
    2017-05-25 11:01:29.577 21880 TRACE nova.compute.manager [instance: 2c5a8e62-62d0-430d-8747-795350bb6939] Exit code: 1
    2017-05-25 11:01:29.577 21880 TRACE nova.compute.manager [instance: 2c5a8e62-62d0-430d-8747-795350bb6939] Stdout: u''
    2017-05-25 11:01:29.577 21880 TRACE nova.compute.manager [instance: 2c5a8e62-62d0-430d-8747-795350bb6939] Stderr: u'qemu-img: error while writing sector 1569792: No space left on device\n'
    2017-05-25 11:01:29.577 21880 TRACE nova.compute.manager [instance: 2c5a8e62-62d0-430d-8747-795350bb6939]
    2017-05-25 11:01:29.580 21880 INFO nova.compute.manager [req-9fa74abf-bcc1-4b7e-aaef-2e17b593a356 6aa5df16b47442c58efde791abd60497 66458b9ead64409fb9d2e0f2c6d67d39 - - -] [instance: 2c5a8e62-62d0-430d-8747-795350bb6939] Terminating instance


    # df -h

Found that the disk is running out

Countermeasure: clean up the disk

# Evacuation failed

Executed a nova evacuation command, but the evacuation failed, nova show reported this error: the state of the shared storage is wrong. That is, the nova instances directory does not have shared storage. If the ceph shared storage is not configured, the shared storage parameters are still included when the nova evacuate command is executed.

    | fault                                | {"message": "Invalid state of instance files on shared storage", "code": 500, "details": "  File \"/usr/lib/python2.7/site-packages/nova/compute/manager.py\", line 354, in decorated_function |
    |                                      |     return function(self, context, *args, **kwargs)                                                                                                                                            |
    |                                      |   File \"/usr/lib/python2.7/site-packages/nova/compute/manager.py\", line 3031, in rebuild_instance                                                                                            |
    |                                      |     _(\"Invalid state of instance files on shared\"                                                                                                                                            |
    |                                      | ", "created": "2017-05-17T01:25:17Z"}       

# There is a certain probability that the console cannot be displayed

Operation steps: console-virtual machine, click the name of the virtual machine, click [console]

Expected result: display the console page normally

Actual result: There is a certain probability that the page will prompt "Failed to connect to server", click to open in a new window to open the console page

The configuration problem of the nova control node, the hosts configuration of memcache and rabbitmq in the configuration file is incorrect

# On the dashboard interface, use the image centos7, flavor-2 to create a virtual machine error. And the error thrown is no host, that is, no suitable scheduling node was found.

If the size of the flavor is smaller than the image requirement, an error will be reported.

But once again, the above conditions are met and an error is reported.

It is possible that the flavor was created with illegal extra_specs

    OS-FLV-DISABLED:disabled  False
    OS-FLV-EXT-DATA:ephemeral   0
    disk    20
    extra_specs {"xxx": "123", "yyy": "321"}

The filtering option turns on the ComputeCapabilitiesFilter filter.

# import error

glance register.log error:

    2017-05-08 03:18:55.890 3185 ERROR glance.common.config [-] Unable to load glance-registry-keystone from configuration file /usr/share/glance/glance-registry-dist-paste.ini.
    Got: ImportError('No module named simplegeneric',)

/usr/lib/python2.7/site-packages/simplegeneric.py has no read permission, who changed it

# keystone error

Permission denied: AH00072: make_sock: could not bind to address [::]:5000

    [root@controller0 ~]# systemctl start httpd.service
    Job for httpd.service failed because the control process exited with error code. See "systemctl status httpd.service" and "journalctl -xe" for details.
    [root@controller0 ~]# systemctl status httpd.service
    ● httpd.service - The Apache HTTP Server
    Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; vendor preset: disabled)
    Active: failed (Result: exit-code) since Sat 2016-05-28 20:22:34 EDT; 11s ago
        Docs: man:httpd(8)
            man:apachectl(8)
    Process: 4501 ExecStop=/bin/kill -WINCH ${MAINPID} (code=exited, status=1/FAILURE)
    Process: 4499 ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND (code=exited, status=1/FAILURE)
    Main PID: 4499 (code=exited, status=1/FAILURE)

    May 28 20:22:34 controller0 httpd[4499]: (13)Permission denied: AH00072: make_sock: could not bind to address [::]:5000
    May 28 20:22:34 controller0 httpd[4499]: (13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:5000

It may be a firewall configuration problem or selinux. If there is no problem with the firewall, check selinux

check selinux status:
1 [root@controller0 ~]# getenforce
2 enforcing #If it is not disabled, it means that selinux is running normally

SELINUX=enforcing changed to selinux=distabled

restart reboot

# re-scheduled: Not authorized for image

The CLI reports an error:

    | fault                                | {"message": "Build of instance d5739cf7-9830-47fd-9a75-e9b1cb4bb421 was re-scheduled: Not authorized for image dcd85799-92f6-4294-91ec-48670a218651.", "code": 500, "details": "  File \"/usr/lib/python2.7/site-packages/nova/compute/manager.py\", line 2258, in _do_build_and_run_instance |

Logging in to the computing node also reports an error

    2017-05-18 15:01:24.867 40639 TRACE nova.compute.manager [instance: d5739cf7-9830-47fd-9a75-e9b1cb4bb421] ImageNotAuthorized: Not authorized for image dcd85799-92f6-4294-91ec-48670a218651.
    2017-05-18 15:01:24.867 40639 TRACE nova.compute.manager [instance: d5739cf7-9830-47fd-9a75-e9b1cb4bb421]

So add

    [DEFAULT]
    auth_strategy=keyston in /etc/nova/nova.conf
    and the missing e
    should be:
    [DEFAULT]
    auth_strategy=keystone

# libvirtError: unsupported configuration: IDE controllers are unsupported for this QEMU binary or machine type

    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f] Traceback (most recent call last):
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]   File "/usr/lib/python2.7/site-packages/nova/compute/manager.py", line 2483, in _build_resources
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]     yield resources
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]   File "/usr/lib/python2.7/site-packages/nova/compute/manager.py", line 2355, in _build_and_run_instance
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]     block_device_info=block_device_info)
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]   File "/usr/lib/python2.7/site-packages/nova/virt/libvirt/driver.py", line 2704, in spawn
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]     block_device_info=block_device_info)
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]   File "/usr/lib/python2.7/site-packages/nova/virt/libvirt/driver.py", line 4758, in _create_domain_and_network
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]     power_on=power_on)
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]   File "/usr/lib/python2.7/site-packages/nova/virt/libvirt/driver.py", line 4689, in _create_domain
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]     LOG.error(err)
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]   File "/usr/lib/python2.7/site-packages/oslo_utils/excutils.py", line 85, in __exit__
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]     six.reraise(self.type_, self.value, self.tb)
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]   File "/usr/lib/python2.7/site-packages/nova/virt/libvirt/driver.py", line 4679, in _create_domain
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]     domain.createWithFlags(launch_flags)
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]   File "/usr/lib/python2.7/site-packages/eventlet/tpool.py", line 186, in doit
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]     result = proxy_call(self._autowrap, f, *args, **kwargs)
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]   File "/usr/lib/python2.7/site-packages/eventlet/tpool.py", line 144, in proxy_call
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]     rv = execute(f, *args, **kwargs)
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]   File "/usr/lib/python2.7/site-packages/eventlet/tpool.py", line 125, in execute
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]     six.reraise(c, e, tb)
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]   File "/usr/lib/python2.7/site-packages/eventlet/tpool.py", line 83, in tworker
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]     rv = meth(*args, **kwargs)
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]   File "/usr/lib64/python2.7/site-packages/libvirt.py", line 1065, in createWithFlags
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f]     if ret == -1: raise libvirtError ('virDomainCreateWithFlags() failed', dom=self)
    2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f] libvirtError: unsupported configuration: IDE controllers are unsupported for this QEMU binary or machine type

Change the IDE to virtio

# qemu-kvm: Cirrus VGA not available

    | fault                                | {"message": "Build of instance a1feb48a-b5f5-48ab-93a7-838bb46573fb was re-scheduled: internal error: process exited while connecting to monitor: 2017-05-18T10:33:34.222333Z qemu-kvm: Cirrus VGA not available", "code": 500, "details": "  File \"/usr/lib/python2.7/site-packages/nova/compute/manager.py\", line 2258, in _do_build_and_run_instance |

Machines with power architecture do not support Cirrus VGA

# The default route cannot be deleted

There are two services to confirm whether the status is normal. network and NetworkManager


# dashboard access lag

The reason for locating the dashboard card is that it should be a nova card,

nova freezes because nova cannot establish a connection with memcached,

It is further located that the default maximum number of connections for memcached is 1024, which has reached the maximum number of connections so far.

The solution is to edit /etc/sysconfig/memcached

The parameters are modified to:

    PORT="11211"
    USER="memcached"
    MAXCONN="65536"
    CACHESIZE="1024"
    OPTIONS=""

restart memcached

# task_state has been in scheduling

Nova boot executes a certain node to generate vm, task_state is always in scheduling, vm_state is always in building, it may be that the nova-compute state of the node that is forced to be scheduled is down.

# Access denied for user 'nova'@'%' to database 'nova_api'


Initialize the database when initializing the nova_api database

    su -s /bin/sh -c "nova-manage api_db sync" nova

Error:

    Access denied for user 'nova'@'%' to database 'nova_api'

The root user enters the database and executes

    > GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY '2267593eb27be7c414fc';

solve

# All nodes have insufficient disks, and return 0 available hosts

Go to the node df -h and find that there is a lot of disk space left. Nova judges the disk space of a node not based on ll, but based on the space occupied by vm and other data.

Such as:


    [root@NFJD-PSC-IBMN-SV356 10d31bfa-961d-44ea-b554-7575315a8e2e]# ll -h
    total 679M
    -rw-rw---- 1 root root  16K Nov 17 17:32 console.log
    -rw-r--r-- 1 root root 679M Nov 18 10:07 disk
    -rw-r--r-- 1 qemu qemu 422K Nov 17 17:24 disk.config
    -rw-r--r-- 1 nova nova  162 Nov 17 17:24 disk.info
    -rw-r--r-- 1 nova nova 3.4K Nov 17 17:24 libvirt.xml
    [root@NFJD-PSC-IBMN-SV356 10d31bfa-961d-44ea-b554-7575315a8e2e]# qemu-img info disk
    image: disk
    file format: qcow2
    virtual size: 40G (42949672960 bytes)
    disk size: 678M
    cluster_size: 65536
    backing file: /var/lib/nova/instances/_base/af019e4c89c44506c068ada379c040848416510e
    Format specific information:
        compat: 1.1
        lazy refcounts: false


ll The output of this file is more than 600 megabytes, but nova statistics are based on 40G. Because the nova code counts disk_over_committed. nova will store the statistical disk information in the table compute_nodes of the nova database and provide it to the scheduler.

The virtual size here minus the disk size is over_commit_size.

It can be seen that only the image in qcow2 format is overcommited here, and the over_commit_size of other files is equal to 0.

In the DiskFilter of the nova scheduling service, disk_allocation_ratio is used to over-allocate disk resources. It is not the same concept as overcommit here. It is the overuse seen from the perspective of the control node, but cannot be seen by the computing node. Overcommit is a calculation The node sees the result obtained after the disk qcow2 compression format, and the remaining space it finally reports is the actual result after deducting the hypothetical qcow2 image file decompression. Therefore, the remaining space actually reported is smaller than the space seen by the naked eye.

If the administrator specifies a computing node during deployment, the virtual machine will be forced to the computing node without going through the scheduling process, forcibly occupying the space that has been included in the oversubscription plan, which may eventually cause the disk resources reported by the computing node is a negative number. And in the future, as the actual disk space occupied by the virtual machine becomes larger and larger, the hard disk space of the computing node may eventually be insufficient.

# novnc can not open the problem location
Maybe the compute firewall has been changed

Add in /etc/sysconfig/iptables

    -A INPUT -p tcp --dport 5900:6100 -j ACCEPT




# qemu-ga fails to start because it cannot find the corresponding virtual serial character device, prompting that the channel cannot be found

glance image-update –property hw_qemu_guest_agent=yes $IMAGE_ID# … For other property configurations, be sure to set property hw_qemu_guest_agent=yes, otherwise libvert will not generate qemu-ga configuration items when starting the virtual machine, causing the qemu-ga inside the virtual machine to fail to find The corresponding virtual serial character device fails to start, prompting that the channel cannot be found.
