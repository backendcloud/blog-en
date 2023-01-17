release time :2018-04-14 05:18

henomenon: openstack dashboard novnc can not view, report Failed to connect to server (code: 1006) error

Check the log:

The consoleauth log information of the 3 controller nodes is as follows:

token expired or false

According to the vncproxy running process below, it should be that the token of the nova-consoleauth cache in step 10 and the check token in step 14 are wrong.

Features of VNC Proxy:
* Separate the public network from the private network
* VNC client runs on the public network, VNCServer runs on the private network, and VNC Proxy acts as an intermediate bridge to connect the two
* VNC Proxy authenticates VNC Client through token
* VNC Proxy not only makes private network access more secure, but also separates the implementation of specific VNC Servers, which can support VNC Servers of different Hypervisors without affecting user experience

Deployment of VNC Proxy

* Deploy the nova-consoleauth process on the Controller node for Token verification
* Deploy the nova-novncproxy service on the Controller node, and the user's VNC Client will directly connect to this service
* The Controller node generally has two network cards, connected to two networks, one is used for external access, we call it public network, or API network, the IP address of this network card is the external network IP, as shown in the figure 172.24.1.1, in addition A network card is used for communication between the various modules of openstack, called the management network, usually the intranet IP, as shown in the figure 10.10.10.2
* Deploy nova-compute on the Compute node with the following configuration in the nova.conf file
  * vnc_enabled=True
  * vncserver_listen=0.0.0.0 //VNC Server listening address
  * vncserver_proxyclient_address=10.10.10.2 //nova vnc proxy accesses the vnc server through the intranet IP, so nova-compute will tell vnc proxy to use this IP to connect to me.
  * novncproxy_base_url= http://172.24.1.1:6080/vnc_auto.html //This url is the url returned to the customer, so the IP inside is the external network IP

The running process of VNC Proxy:
1. A user tries to open the VNC Client connected to the virtual machine from the browser
2. The browser sends a request to nova-api, requesting to return the url to access vnc
3. nova-api calls the get vnc console method of nova-compute, requesting Return the information about connecting to VNC
4. nova-compute calls the get vnc console function of libvirt
5. libvirt will obtain the information of VNC Server by parsing the /etc/libvirt/qemu/instance-0000000c.xml file running on the virtual machine
6. libvirt will Host, port and other information are returned to nova-compute in json format 7. nova-compute will randomly generate a UUID as a Token 8. nova-compute synthesizes the information returned by libvirt and the information in the configuration file into connect_info and returns it to nova-api
7. nova-api will call the authorize_console function of nova-
consoleauth 10.nova-consoleauth will cache the information of instance –> token, token –> connect_info
8. nova-api will return the access url information in connect_info to the browser: http ://172.24.1.1:6080/vnc_auto.html?token=7efaee3f-eada-4731-a87c-e173cbd25e98&title=helloworld%289169fdb2-5b74-46b1-9803-60d2926bd97c%29
9. The browser will try to open this link
10. This link will send the request to nova-novncproxy
11. nova-novncproxy calls the check_token function of nova-consoleauth 15.nova-consoleauth
verifies the token and returns the connect_info corresponding to the instance Give nova-novncproxy
12. nova-novncproxy connects to the VNC Server on the compute node through the host, port and other information in connect_info, thus starting the work of the proxy

It may be that the high-availability deployment of the control node may be that the memcache is not configured or configured incorrectly.

The investigation found that the configuration item was wrong with a letter . After changing

the configuration memcached_serversitem in /etc/nova/nova.conf to novnc memcache_servers, it can be accessed. , and changed to P version
memcached_memcache_

    Option memcached_servers is deprecated in Mitaka. Operators should use oslo.cache configuration instead. Specifically enabled option under [cache] section should be set to True and the url(s) for the memcached servers should be in [cache]/memcache_servers option.

    https://docs.openstack.org/oslo.cache/1.16.0/opts.html
    memcache_servers
    Type:   list
    Default:    localhost:11211
    Memcache servers in the format of “host:port”. (dogpile.cache.memcache and oslo_cache.memcache_pool backends only).

