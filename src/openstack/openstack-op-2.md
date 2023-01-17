release time :2018-08-28 14:48

# Start openvswitch error

    [root@SRIOV03 ml2]#  systemctl start openvswitch.service
    A dependency job for openvswitch.service failed. See 'journalctl -xe' for details.
    [root@SRIOV03 ml2]# journalctl -xe
    -- Defined-By: systemd
    -- Support: http://lists.freedesktop.org/mailman/listinfo/systemd-devel
    -- 
    -- Unit ovsdb-server.service has begun starting up.
    Aug 28 10:33:00 SRIOV03 runuser[6958]: PAM audit_open() failed: Permission denied
    Aug 28 10:33:00 SRIOV03 ovs-ctl[6929]: runuser: System error
    Aug 28 10:33:00 SRIOV03 ovs-ctl[6929]: /etc/openvswitch/conf.db does not exist ... (warning).
    Aug 28 10:33:00 SRIOV03 runuser[6960]: PAM audit_open() failed: Permission denied
    Aug 28 10:33:00 SRIOV03 ovs-ctl[6929]: Creating empty database /etc/openvswitch/conf.db runuser: System error
    Aug 28 10:33:00 SRIOV03 ovs-ctl[6929]: [FAILED]
    Aug 28 10:33:00 SRIOV03 systemd[1]: ovsdb-server.service: control process exited, code=exited status=1
    Aug 28 10:33:00 SRIOV03 systemd[1]: Failed to start Open vSwitch Database Unit.

It may be that selinux is active, turn it off

    # vim /etc/selinux/config 
    SELINUX=enforcing -> SELINUX=disabled
    # setenforce 0
    # getenforce 
    Permissive
    # systemctl restart openvswitch.service

# Generate vm and report ethtool ens4f0 error

    2018-08-28 11:19:51.568 8034 DEBUG oslo_concurrency.processutils [req-cfeb820d-1167-4c32-a336-8fed1120556b - - - - -] u'ethtool ens4f0' failed. Not Retrying. execute /usr/lib/python2.7/site-packages/oslo_concurrency/processutils.py:452
    2018-08-28 11:19:51.568 8034 WARNING nova.virt.extrautils [req-cfeb820d-1167-4c32-a336-8fed1120556b - - - - -] When exec ethtool network card, exception occurs: Unexpected error while running command.

    /etc/nova/nova.conf
    nw_interface_name = ens4f0

Just change it to a data plane network card



# Create a vm and report qemu unexpectedly closed the monitor error

    | fault                                | {"message": "internal error: qemu unexpectedly closed the monitor: 2018-08-28T04:08:03.821855Z qemu-kvm: -chardev pty,id=charserial0,logfile=/dev/fdset/2,logappend=on: char device redirected to /dev/pts/3 (label charserial0) |
    |                                      | 2018-08-28T04:08:04.024833Z qemu-kvm: -vnc ", "code": 500, "details": "  File \"/usr/lib/python2.7/site-packages/nova/compute/manager.py\", line 1856, in _do_build_and_run_instance                                             |
    |                                      |     filter_properties)                                                                                                                                                                                                           |
    |                                      |   File \"/usr/lib/python2.7/site-packages/nova/compute/manager.py\", line 2086, in _build_and_run_instance                                                                                                                       |
    |                                      |     instance_uuid=instance.uuid, reason=six.text_type(e))  

```bash
# vim /var/log/libvirt/libvirtd.log 
2018-08-28 04:29:31.302+0000: 11230: error : virNetSocketReadWire:1808 : End of file while reading data: Input/output error
2018-08-28 04:36:25.462+0000: 11235: info : qemuDomainDefineXMLFlags:7286 : Creating domain 'instance-000007bc'
2018-08-28 04:36:25.779+0000: 11230: error : qemuMonitorIORead:588 : Unable to read from monitor: Connection reset by peer
2018-08-28 04:36:25.780+0000: 11230: error : qemuProcessReportLogError:1862 : internal error: qemu unexpectedly closed the monitor: 2018-08-28T04:36:25.720940Z qemu-kvm: -chardev pty,id=charserial0,logfile=/dev/fdset/2,logappend=on: char device redirected to /dev/pts/2 (label charserial0)
2018-08-28T04:36:25.778613Z qemu-kvm: -vnc 10.144.85.92:0: Failed to start VNC server: Failed to bind socket: Cannot assign requested address
2018-08-28 04:36:26.148+0000: 11239: info : qemuDomainUndefineFlags:7401 : Undefining domain 'instance-000007bc'
```

Because the vnc ip configuration error in nova.conf

    [vnc]
    enabled = True
    novncproxy_host = 10.144.85.92
    vncserver_proxyclient_address = 10.144.85.92
    vncserver_listen = 10.144.85.92
    novncproxy_base_url = http://10.144.85.238:6080/vnc_auto.html 

# The vm can be generated on the new node, but the ip cannot be obtained when logging in to the vm

It may be that the switch vlan is not allowed

# Create virtual machines concurrently, some fail

The error seen in the log is that rabbitmq cannot connect to the port

It is initially determined that rabbitmq cannot work normally due to excessive pressure on rabbitmq. Take some steps now to improve the situation.

①. Change the rabbitmq node 2 and node 3 of the rabbitmq cluster from disc mode to ram mode.

②Distribute the rabbitmq pressure to 3 rabbitmq nodes. By looking at the rabbitmq log, it is found that before the modification, the rabbitmq pressure is mainly on node01, and the other two nodes will hardly process message requests.

③ Turn off the rate mode function of rabbitmq monitoring. The rate mode function monitors the message output rate in the message queue. Closing has no effect on services.

④ [glance] api_servers=vip:9292 configured before nova-compute, vip is the address of the management network, when creating virtual machines and other concurrency, there will be image downloads occupying the management network channel, causing network sensitive service messages such as rabbitmq to be blocked, and even messages Timeout, therefore, configure its api_servers as the storage address of the control node, and change the mirror service of the control node from monitoring the management network to monitoring the storage network.

After taking the above measures, a test was carried out

1. Tested that 100 virtual machines were created concurrently, and none of the virtual machines failed.
2. Tested to create 100 volumes concurrently, none of them failed

# A control node memory usage is too high alarm

A control node has a high memory usage alarm. It is found that the rabbitmq process is abnormal. The backlog of messages in the message queue causes the memory to increase and cannot be released. Restart the rabbitmq process to solve the problem. To solve the problem, the rabbitmq configuration file needs to be modified to make the backlog The messages are stored on disk rather than in memory.

# The resize operation on the cloud host is not successful

Resize a vm, that is, change from a small flavor to a large flavor, without success

Check the nova-compute.log of the node where the cloud host is located

    2017-07-03 17:50:08.573 24296 TRACE oslo_messaging.rpc.dispatcher   File "/usr/lib/python2.7/site-packages/nova/compute/manager.py", line 6959, in _error_out_instance_on_exception
    2017-07-03 17:50:08.573 24296 TRACE oslo_messaging.rpc.dispatcher     raise error.inner_exception
    2017-07-03 17:50:08.573 24296 TRACE oslo_messaging.rpc.dispatcher ResizeError: Resize error: not able to execute ssh command: Unexpected error while running command.
    2017-07-03 17:50:08.573 24296 TRACE oslo_messaging.rpc.dispatcher Command: ssh 172.16.231.26 -p 22 mkdir -p /var/lib/nova/instances/54091e90-55f5-4f4d-8f66-31fbc787584f
    2017-07-03 17:50:08.573 24296 TRACE oslo_messaging.rpc.dispatcher Exit code: 255
    2017-07-03 17:50:08.573 24296 TRACE oslo_messaging.rpc.dispatcher Stdout: u''
    2017-07-03 17:50:08.573 24296 TRACE oslo_messaging.rpc.dispatcher Stderr: u'@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@\r\n@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @\r\n@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@\r\nIT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!\r\nSomeone could be eavesdropping on you right now (man-in-the-middle attack)!\r\nIt is also possible that a host key has just been changed.\r\nThe fingerprint for the ECDSA key sent by the remote host is\nc8:0b:98:9d:8d:d8:14:89:e6:fe:97:66:22:4e:57:13.\r\nPlease contact your system administrator.\r\nAdd correct host key in /var/lib/nova/.ssh/known_hosts to get rid of this message.\r\nOffending ECDSA key in /var/lib/nova/.ssh/known_hosts:34\r\nPassword authentication is disabled to avoid man-in-the-middle attacks.\r\nKeyboard-interactive authentication is disabled to avoid man-in-the-middle attacks.\r\nPermission denied (publickey,gssapi-keyex,gssapi-with-mic,password).\r\n'


Analyze the reasons

It may be that the host key of the nova user of the computing node has changed

Countermeasures

For all computing nodes, delete /var/lib/nova/.ssh/known_hosts rows related to computing nodes, log in to each other with nova user ssh again, add a host key and ensure that ssh between nova user computing nodes is mutual trust

# After the ceilometer is installed, start the service and the status is wrong

    [root@NFJD-TESTN-COMPUTE-2 ceilometer-2015.1.5]# systemctl restart openstack-ceilometer-compute.service
    [root@NFJD-TESTN-COMPUTE-2 ceilometer-2015.1.5]# systemctl status openstack-ceilometer-compute.service
    ● openstack-ceilometer-compute.service - OpenStack ceilometer compute agent
    Loaded: loaded (/usr/lib/systemd/system/openstack-ceilometer-compute.service; enabled; vendor preset: disabled)
    Active: failed (Result: exit-code) since Fri 2016-12-23 09:39:11 CST; 2s ago
    Process: 8420 ExecStart=/usr/bin/ceilometer-agent-compute --logfile /var/log/ceilometer/compute.log (code=exited, status=1/FAILURE)
    Main PID: 8420 (code=exited, status=1/FAILURE)

    Dec 23 09:39:11 NFJD-TESTN-COMPUTE-2 ceilometer-agent-compute[8420]: File "/usr/lib/python2.7/site-packages/ceilometer/cmd/polling.py", line 82, in main_compute
    Dec 23 09:39:11 NFJD-TESTN-COMPUTE-2 ceilometer-agent-compute[8420]: service.prepare_service()
    Dec 23 09:39:11 NFJD-TESTN-COMPUTE-2 ceilometer-agent-compute[8420]: File "/usr/lib/python2.7/site-packages/ceilometer/service.py", line 117, in prepare_service
    Dec 23 09:39:11 NFJD-TESTN-COMPUTE-2 ceilometer-agent-compute[8420]: cfg.CONF(argv[1:], project='ceilometer', validate_default_values=True)
    Dec 23 09:39:11 NFJD-TESTN-COMPUTE-2 ceilometer-agent-compute[8420]: File "/usr/lib/python2.7/site-packages/oslo_config/cfg.py", line 1860, in __call__
    Dec 23 09:39:11 NFJD-TESTN-COMPUTE-2 ceilometer-agent-compute[8420]: self._namespace._files_permission_denied)
    Dec 23 09:39:11 NFJD-TESTN-COMPUTE-2 ceilometer-agent-compute[8420]: oslo_config.cfg.ConfigFilesPermissionDeniedError: Failed to open some config files: /etc/ceilometer/ceilometer.conf
    Dec 23 09:39:11 NFJD-TESTN-COMPUTE-2 systemd[1]: openstack-ceilometer-compute.service: main process exited, code=exited, status=1/FAILURE
    Dec 23 09:39:11 NFJD-TESTN-COMPUTE-2 systemd[1]: Unit openstack-ceilometer-compute.service entered failed state.
    Dec 23 09:39:11 NFJD-TESTN-COMPUTE-2 systemd[1]: openstack-ceilometer-compute.service failed.

The conf file has no permission, modify the permission, and the status is ok. There is no permission because the root user scp the configuration file from another available computing node.

    [root@NFJD-TESTN-COMPUTE-2 ceilometer-2015.1.5]# chown -R ceilometer:ceilometer /etc/ceilometer
    [root@NFJD-TESTN-COMPUTE-2 ceilometer]# systemctl restart openstack-ceilometer-compute.service
    [root@NFJD-TESTN-COMPUTE-2 ceilometer]# systemctl status openstack-ceilometer-compute.service
    ● openstack-ceilometer-compute.service - OpenStack ceilometer compute agent
    Loaded: loaded (/usr/lib/systemd/system/openstack-ceilometer-compute.service; enabled; vendor preset: disabled)
    Active: active (running) since Fri 2016-12-23 09:42:32 CST; 2s ago
    Main PID: 9459 (ceilometer-agen)
    CGroup: /system.slice/openstack-ceilometer-compute.service
            └─9459 /usr/bin/python /usr/bin/ceilometer-agent-compute --logfile /var/log/ceilometer/compute.log

    Dec 23 09:42:32 NFJD-TESTN-COMPUTE-2 systemd[1]: Started OpenStack ceilometer compute agent.
    Dec 23 09:42:32 NFJD-TESTN-COMPUTE-2 systemd[1]: Starting OpenStack ceilometer compute agent...

> file permission problem, if the configuration file has no configuration file permission after replacement, for example, the file owner who was originally root:nova is replaced by root:root, there will be problems that the service cannot run normally.

# The version is too new to cause an error in referencing the package

    | fault                                | {"message": "Build of instance 15d9db88-d0a9-40a8-83e9-9ede3001b112 was re-scheduled: 'module' object has no attribute 'to_utf8'", "code": 500, "details": "  File \"/usr/lib/python2.7/site-packages/nova/compute/manager.py\", line 2258, in _do_build_and_run_instance |


    2017-05-25 11:30:34.529 32687 TRACE nova.compute.manager [instance: 15d9db88-d0a9-40a8-83e9-9ede3001b112]   File "/usr/lib/python2.7/site-packages/nova/utils.py", line 207, in execute
    2017-05-25 11:30:34.529 32687 TRACE nova.compute.manager [instance: 15d9db88-d0a9-40a8-83e9-9ede3001b112]     return processutils.execute(*cmd, **kwargs)
    2017-05-25 11:30:34.529 32687 TRACE nova.compute.manager [instance: 15d9db88-d0a9-40a8-83e9-9ede3001b112]   File "/usr/lib64/python2.7/site-packages/oslo_concurrency/processutils.py", line 287, in execute
    2017-05-25 11:30:34.529 32687 TRACE nova.compute.manager [instance: 15d9db88-d0a9-40a8-83e9-9ede3001b112]     process_input = encodeutils.to_utf8(process_input)
    2017-05-25 11:30:34.529 32687 TRACE nova.compute.manager [instance: 15d9db88-d0a9-40a8-83e9-9ede3001b112] AttributeError: 'module' object has no attribute 'to_utf8'
    2017-05-25 11:30:34.529 32687 TRACE nova.compute.manager [instance: 15d9db88-d0a9-40a8-83e9-9ede3001b112]
    2017-05-25 11:30:34.531 32687 INFO nova.compute.manager [req-22c24896-bcad-4464-a2ab-c019c45ac0e3 2a2a6d4700034e53a198dac0a71aab6e fe3e50d5fc274d29948ff7fd46b214dc - - -] [instance: 15d9db88-d0a9-40a8-83e9-9ede3001b112] Terminating instance


    [root@compute-power ~]# vim /usr/lib64/python2.7/site-packages/oslo_concurrency/processutils.py
    284     cwd = kwargs.pop('cwd', None)
    285     process_input = kwargs.pop('process_input', None)
    286     if process_input is not None:
    287         process_input = encodeutils.to_utf8(process_input)


    [root@compute-power ~]# pip show oslo.concurrency
    Name: oslo.concurrency
    Version: 3.20.0
    Summary: Oslo Concurrency library
    Home-page: http://docs.openstack.org/developer/oslo.concurrency
    Author: OpenStack
    Author-email: openstack-dev@lists.openstack.org
    License: UNKNOWN
    Location: /usr/lib64/python2.7/site-packages
    Requires: pbr, oslo.config, fasteners, oslo.i18n, oslo.utils, six, enum34


    [root@bc36-compute-3 oslo_concurrency]# pip show oslo.concurrency
    ---
    Name: oslo.concurrency
    Version: 1.8.2
    Location: /usr/lib/python2.7/site-packages
    Requires: 
    ...

The oslo.concurrency version is too new, so replace it with an old version to solve the problem.


# vm error

Error message:

    Flavor's disk is too small for requested image. Flavor disk is 16106127360 bytes, image is 21474836480 bytes.].

It means that the disk space of the Flavor (cloud host type) used when creating the vm does not meet the image mirroring requirements!

# No valid host was found

2016-11-01 01:28:38.889 51843 WARNING nova.scheduler.utils [req-9eb2b8ec-216b-4073-95bd-1fbb51844faf 52ba7917bb284af7ad6ac313b7e8e948 0cd3632df93d48d6b2c24c67f70e56b8 - - -] Failed to compute_task_build_instances: No valid host was found. There are not enough hosts available.

The main reasons for this problem are:
* It is caused by insufficient memory, insufficient CPU resources, and insufficient hard disk space resources of the computing node; if the type and specification of the cloud host are reduced, it can be found that the creation will be successful.
* The network configuration is incorrect, resulting in the failure to obtain the ip when creating a virtual machine; the network is unreachable or caused by a firewall.
* The openstack-nova-compute service status problem. You can try to restart the nova-related services of the control node and the openstack-nova-compute service of the computing node; check the nova.conf configuration of the control node and computing node for improper configuration.
* There are many reasons for this error report. For details, check the detailed analysis of the logs under /var/log/nova. The focus is nova-compute.log, nova-conductor.log logs


# In the logs of nova-scheduler and nova-compute, you can see the error message "ImageNotFound: Image 37aaedc7-6fe6-4fc8-b110-408d166b8e51 could not be found"!

    root@controller ~]# /etc/init.d/openstack-glance-api status
    openstack-glance-api (pid  2222) is running...
    [root@controller ~]# /etc/init.d/openstack-glance-registry status
    openstack-glance-registry (pid  2694) is running...


normal status

    [root@controller ~]# glance image-list
    +--------------------------------------+---------------+-------------+------------------+-------------+--------+
    | ID                                   | Name          | Disk Format | Container Format | Size        | Status |
    +--------------------------------------+---------------+-------------+------------------+-------------+--------+
    | 37aaedc7-6fe6-4fc8-b110-408d166b8e51 | cirrors       | qcow2       | bare             | 13200896    | active |
    +--------------------------------------+---------------+-------------+------------------+-------------+--------+

It works normally, try to upload a mirror image, and you can also create a vm normally, what is the reason? ?

Because in the operation and maintenance process, the default path of the modified glance is changed from /var/lib/glance/images to /data1/glance, and the images under /var/lib/glance/images are mv to /data1/glance At this time, although the data has passed before, the metadata information of the image is firmly recorded in the image_locations table of glance. Check it out:

    mysql> select * from glance.image_locations where image_id='37aaedc7-6fe6-4fc8-b110-408d166b8e51'\G;                                                                        
    *************************** 1. row ***************************
            id: 37
    image_id: 37aaedc7-6fe6-4fc8-b110-408d166b8e51
        value: file:///var/lib/glance/images/37aaedc7-6fe6-4fc8-b110-408d166b8e51    #元凶
    created_at: 2015-12-21 06:10:24
    updated_at: 2015-12-21 06:10:24
    deleted_at: NULL
    deleted: 0
    meta_data: {}
        status: active
    1 row in set (0.00 sec)

Real image: The images in the original directory /var/lib/glance/images have been mv'd to /data1/glance, but the original path content is still recorded in the database. Therefore, a problem arises: when nova When trying to start an instance, nova will go to the instance image cache path, and check whether there is the image in /var/lib/nova/_base by default, and if not, send a result api request to glance to request to download the image of the specified image to the local , glance looks for the image according to the value defined by image_locations in the database, which leads to failure!
Solution: update the metadata information of glance

    mysql> update glance.image_locations set value='file:///data1/glance/37aaedc7-6fe6-4fc8-b110-408d166b8e51' where image_id='37aaedc7-6fe6-4fc8-b110-408d166b8e51'\G;             
    Query OK, 1 row affected (0.05 sec)
    Rows matched: 1  Changed: 1  Warnings: 0

Rebuild the virtual machine, the problem is solved! ! !

# Use config drive to set ip, resulting in ip conflict cannot create vm

    2017-06-22 11:58:50.819 18449 INFO nova.virt.libvirt.driver [req-4edc310f-8382-43ce-a021-8f04c55d5a51 25380a8cde624ae2998224074d4092ee 1cca7b8b0e0d47cb8d6eda06837960ed - - -] [instance: fb28867f-da0f-48f9-b8f8-33704c845117] Using config drive
    ...
    2017-06-22 11:59:16.686 18449 TRACE nova.compute.manager [instance: fb28867f-da0f-48f9-b8f8-33704c845117] FixedIpAlreadyInUse: Fixed IP 172.16.1.4 is already in use.
    2017-06-22 11:59:16.686 18449 TRACE nova.compute.manager [instance: fb28867f-da0f-48f9-b8f8-33704c845117]
    2017-06-22 11:59:16.690 18449 INFO nova.compute.manager [req-4edc310f-8382-43ce-a021-8f04c55d5a51 25380a8cde624ae2998224074d4092ee 1cca7b8b0e0d47cb8d6eda06837960ed - - -] [instance: fb28867f-da0f-48f9-b8f8-33704c845117] Terminating instance
    2017-06-22 11:59:16.698 18449 INFO nova.virt.libvirt.driver [-] [instance: fb28867f-da0f-48f9-b8f8-33704c845117] During wait destroy, instance disappeared.
    2017-06-22 11:59:16.708 18449 INFO nova.virt.libvirt.driver [req-4edc310f-8382-43ce-a021-8f04c55d5a51 25380a8cde624ae2998224074d4092ee 1cca7b8b0e0d47cb8d6eda06837960ed - - -] [instance: fb28867f-da0f-48f9-b8f8-33704c845117] Deleting instance files /var/lib/nova/instances/fb28867f-da0f-48f9-b8f8-33704c845117_del
    2017-06-22 11:59:16.708 18449 INFO nova.virt.libvirt.driver [req-4edc310f-8382-43ce-a021-8f04c55d5a51 25380a8cde624ae2998224074d4092ee 1cca7b8b0e0d47cb8d6eda06837960ed - - -] [instance: fb28867f-da0f-48f9-b8f8-33704c845117] Deletion of /var/lib/nova/instances/fb28867f-da0f-48f9-b8f8-33704c845117_del complete

# Failed to mount and unmount volume

    2017-05-09 14:19:13.957 12778 TRACE oslo_messaging.rpc.dispatcher VolumeDeviceNotFound: Volume device not found at [u'/dev/disk/by-path/ip-10.11.23.132:3260-iscsi-iqn.2000-09.com.fujitsu:storage-system.eternus-dx400:00C0C1P0-lun-78', u'/dev/disk/by-path/ip-10.11.23.133:3260-iscsi-iqn.2000-09.com.fujitsu:storage-system.eternus-dx400:00C1C1P0-lun-78'].

Checked the route of the node that can be mounted normally, and found that the route of the compute-2 node was lost.

    10.11.23.128    10.11.231.1     255.255.255.128 UG    0      0        0 enp17s0f0.607
    10.11.231.0     0.0.0.0         255.255.255.0   U     0      0        0 enp17s0f0.607

After adding the route, it can be successfully mounted and unmounted

    [root@NFJD-TESTN-COMPUTE-2 ~]# ip a|grep 607
    [root@NFJD-TESTN-COMPUTE-2 ~]# ip link add link enp17s0f0 name enp17s0f0.607 type vlan id 607
    [root@NFJD-TESTN-COMPUTE-2 ~]# ip link set up enp17s0f0.607
    [root@NFJD-TESTN-COMPUTE-2 ~]# ifconfig enp17s0f0 0.0.0.0
    [root@NFJD-TESTN-COMPUTE-2 ~]# ifconfig enp17s0f0.607 10.11.231.26 netmask 255.255.255.0
    [root@NFJD-TESTN-COMPUTE-2 ~]# route add -net 10.11.23.128/25 gw 10.11.231.1 dev enp17s0f0.607
    [root@NFJD-TESTN-COMPUTE-2 ~]# ip a|grep 607
    27: enp17s0f0.607@enp17s0f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1600 qdisc noqueue state UP 
        inet 10.11.231.26/24 brd 10.11.231.255 scope global enp17s0f0.607

# Failed when vm mount volume

The nova-compute.log error of the nova compute node is as follows:

    2017-05-09 15:18:01.952 10888 ERROR nova.virt.block_device [req-344ed875-4bc4-40bf-95e3-a0ff48df6059 62f52135115f4898bd0d82c1f0cd632b 6c149dcd3cf64171b8dd972dd03bbac0 - - -] [instance: ddcf7976-b282-45ac-a167-9865178ed629] Driver failed to attach volume 88442395-87d9-41f1-b9d8-0df4b941846a at /dev/vdc
    2017-05-09 15:18:01.952 10888 TRACE nova.virt.block_device [instance: ddcf7976-b282-45ac-a167-9865178ed629] Traceback (most recent call last):
    2017-05-09 15:18:01.952 10888 TRACE nova.virt.block_device [instance: ddcf7976-b282-45ac-a167-9865178ed629] File "/usr/lib/python2.7/site-packages/nova/virt/block_device.py", line 276, in attach
    2017-05-09 15:18:01.952 10888 TRACE nova.virt.block_device [instance: ddcf7976-b282-45ac-a167-9865178ed629] device_type=self['device_type'], encryption=encryption)
    2017-05-09 15:18:01.952 10888 TRACE nova.virt.block_device [instance: ddcf7976-b282-45ac-a167-9865178ed629] File "/usr/lib/python2.7/site-packages/nova/virt/libvirt/driver.py", line 1210, in attach_volume
    2017-05-09 15:18:01.952 10888 TRACE nova.virt.block_device [instance: ddcf7976-b282-45ac-a167-9865178ed629] self._connect_volume(connection_info, disk_info)
    2017-05-09 15:18:01.952 10888 TRACE nova.virt.block_device [instance: ddcf7976-b282-45ac-a167-9865178ed629] File "/usr/lib/python2.7/site-packages/nova/virt/libvirt/driver.py", line 1157, in _connect_volume
    2017-05-09 15:18:01.952 10888 TRACE nova.virt.block_device [instance: ddcf7976-b282-45ac-a167-9865178ed629] driver.connect_volume(connection_info, disk_info)
    2017-05-09 15:18:01.952 10888 TRACE nova.virt.block_device [instance: ddcf7976-b282-45ac-a167-9865178ed629] File "/usr/lib/python2.7/site-packages/oslo_concurrency/lockutils.py", line 445, in inner
    2017-05-09 15:18:01.952 10888 TRACE nova.virt.block_device [instance: ddcf7976-b282-45ac-a167-9865178ed629] return f(*args, **kwargs)
    2017-05-09 15:18:01.952 10888 TRACE nova.virt.block_device [instance: ddcf7976-b282-45ac-a167-9865178ed629] File "/usr/lib/python2.7/site-packages/nova/virt/libvirt/volume.py", line 505, in connect_volume
    2017-05-09 15:18:01.952 10888 TRACE nova.virt.block_device [instance: ddcf7976-b282-45ac-a167-9865178ed629] if self.use_multipath:
    2017-05-09 15:18:01.952 10888 TRACE nova.virt.block_device [instance: ddcf7976-b282-45ac-a167-9865178ed629] File "/usr/lib/python2.7/site-packages/nova/virt/libvirt/volume.py", line 505, in connect_volume
    2017-05-09 15:18:01.952 10888 TRACE nova.virt.block_device [instance: ddcf7976-b282-45ac-a167-9865178ed629] if self.use_multipath:
    2017-05-09 15:18:01.952 10888 TRACE nova.virt.block_device [instance: ddcf7976-b282-45ac-a167-9865178ed629] File "/usr/lib64/python2.7/bdb.py", line 49, in trace_dispatch
    2017-05-09 15:18:01.952 10888 TRACE nova.virt.block_device [instance: ddcf7976-b282-45ac-a167-9865178ed629] return self.dispatch_line(frame)
    2017-05-09 15:18:01.952 10888 TRACE nova.virt.block_device [instance: ddcf7976-b282-45ac-a167-9865178ed629] File "/usr/lib64/python2.7/bdb.py", line 68, in dispatch_line
    2017-05-09 15:18:01.952 10888 TRACE nova.virt.block_device [instance: ddcf7976-b282-45ac-a167-9865178ed629] if self.quitting: raise BdbQuit
    2017-05-09 15:18:01.952 10888 TRACE nova.virt.block_device [instance: ddcf7976-b282-45ac-a167-9865178ed629] BdbQuit
    2017-05-09 15:18:01.952 10888 TRACE nova.virt.block_device [instance: ddcf7976-b282-45ac-a167-9865178ed629]

Maybe someone added a breakpoint when debugging the code, and forgot to delete the breakpoint
to find it in the code. Sure enough, delete the breakpoint, restart the service, and the test can mount the volume

# A host doesn't support passthrough error occurs when creating a vm

    | fault                                | {"message": "unsupported configuration: host doesn't support passthrough of host PCI devices", "code": 500, "details": "  File \"/usr/lib/python2.7/site-packages/nova/compute/manager.py\", line 1856, in _do_build_and_run_instance |
    |                                      |     filter_properties) 

This error may be caused by the parent VT-d (or IOMMU) not being enabled.
Make sure the "intel_iommu=on" boot parameter has been enabled as described above.

Found that the /etc/default/grub file has been modified

Configure the /etc/default/grub file of the computing node, add intel_iommu=on to GRUB_CMDLINE_LINUX to activate the VT-d function, and restart the physical machine

    $ cat /etc/default/grub
    GRUB_TIMEOUT=5
    GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
    GRUB_DEFAULT=saved
    GRUB_DISABLE_SUBMENU=true
    GRUB_TERMINAL_OUTPUT="console"
    GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=bclinux/root rd.lvm.lv=bclinux/swap intel_iommu=on rhgb quiet"
    GRUB_DISABLE_RECOVERY="true"

But there is no restart, no restart will not take effect

After restarting, the sriov virtual machine can be generated normally

For intel cpu and amd cpu, the grub configuration is different, please refer to the article for specific configuration: http://pve.proxmox.com/wiki/Pci_passthrough

update-grub

After editing the grub file, you need to update

    grub2-mkconfig   # fedora arch centos
    update-grub            # ubuntu debian

Restart the computer for it to take effect

    # cat /proc/cmdline
