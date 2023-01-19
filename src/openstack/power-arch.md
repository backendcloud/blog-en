release time :2018-10-21 16:21

# Target
Install computing components on the power machine, use the power machine as a computing node, and test the basic functions of Openstack.

# environment
1 control node (x86 machine)
5 computing nodes (4 x86 machines, 1 power machine)

# Problems with installing dependencies
Even if the yum source is configured as the yun source of the power architecture, there are still some dependencies that cannot be found.
Power machine installation components and the dependencies required by the components will encounter various problems that yum dependencies cannot find.

Solution:
1) Manual pip installation for python packages regardless of architecture
2) It needs to be compiled into Power architecture compiler. Or search online, or compile and install on power.

# There is a problem with the power machine's support for the IDE
2017-05-18 15:06:09.522 41033 TRACE nova.compute.manager [instance: c348b942-4553-4023-bbcb-296f3b1bf14f] libvirtError: unsupported configuration: IDE controllers are unsupported for this QEMU machine
binary Time to solve by changing the IDE to virtio

# python script execution error
{"message": "Build of instance 15d9db88-d0a9-40a8-83e9-9ede3001b112 was re-scheduled: 'module' object has no attribute 'to_utf8'", "code": 500, "details": "File "/usr /lib/python2.7/site-packages/nova/compute/manager.py", line 2258, in _do_build_and_run_instance
will report an error when executed. Some are because the python package installed by pip automatically installs dependencies, and the relationship between versions is disordered, so it needs to be uninstalled Current version, reinstall the required version.

# vnc can't be used
{"message": "Build of instance a1feb48a-b5f5-48ab-93a7-838bb46573fb was re-scheduled: internal error: process exited while connecting to monitor: 2017-05-18T10:33:34.222333Z qemu-kvm: Cirrus VGA available”, “code”: 500, “details”: “ File "/usr/lib/python2.7/site-packages/nova/compute/manager.py", line 2258, in _do_build_and_run_instance
By modifying the libvirt source code, recompile Solved by the way of installation.

# keyboard input not responding
The virtual machine generated with the ubuntu14.04 image does not respond to keyboard input.
There is no such problem when switching to ubuntu16.04. There must be more problems with Centos6 and 7. The official website of the Centos image file is not downloaded.

# Nova tests several common operations on POWER machines
Tested several common operations of Nova on vm life cycle management, such as generation, restart, suspend, resume, shutdown, delete. The test is OK.

Tested to mount and unmount volumes on vm. The test is OK.

Tested Nova's resource statistics on the power machine, and the monitoring of the heartbeat of the nova service on the power machine. The test is OK.

# in conclusion
Computing components are deployed on power machines. There are many problems, it is difficult to deploy, the operation is too risky, and there are too many uncontrollable factors.

Linux(Centos)+openstack is installed on a power machine, there is no mature solution, or there is no precedent. It costs too much to try and research by yourself. It is professional research and development, and technology is not very likely to go in this direction. Generally, it is left to the community. And Linux (Centos) community fedora, PowerPC is downgraded to a secondary architecture by fedora, after x86 and arm architecture

https://fedoraproject.org/wiki/Architectures/ARM/Planning/Primary

It is assumed that the computing components can run on the power machine intact, but the power architecture machine can only run the power architecture virtual machine, and there are very few customer virtual machines with the power architecture.