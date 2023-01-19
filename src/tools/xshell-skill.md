release time :2016-07-05 22:57

# Paste

For example, openstack vm_id:

	424288e4-23a7-45de-bb5d-742bd6c54561

If the default delimiter is used, double-clicking can only select a part, you need to press and hold the mouse to drag, or more or less, or it will take a little time.

After changing the settings:

In the "Keyboard and Mouse" tab of "Options"

1. Remove the "-" in the delimiter.
2. Check "Automatically copy the selected text to the clipboard" and vm_id can be selected by double-clicking. There is no need to select copy and paste. It has been copied at the same time as it is selected. At this time, only the middle mouse button is needed to complete the paste.

# Split Screen

Only 5.0 or above is supported, and it can be realized by dragging the label to a certain position on the screen with the mouse. It is useful in some cases, such as analyzing networks and comparing jos.


# multi-level jump

In the internal environment of an enterprise, not every node has an external network ip, often through the Jump Server, and then the Jump Server login to other nodes. In a complex environment, there may be more than two levels of login, or even three or more levels of login. It can be easily achieved with xshell.

When creating a new session, or click the properties of the created session, select "Login Script" in "Connection" in "Category"

Select the "Execute the following wait and send rules" check box to activate the two columns below Expect and Send to display interactive functions similar to tcl's expect or python pexpect package.

|Expect   |Send                    |
|---------|------------------------|
|$        |ssh deployer@xx.xx.xx.xx|
|password:|xxxxxx                  |

The above is two levels of login, and more levels of login can be added later.

# tunnel forwarding

Select "Tunnel" in "SSH" in "Connection" in "Category" in the properties of the session.

Two commonly used methods Local (Outgoing) and Dynamic (SOCKS4/5)

Take access to the openstack dashboard on the intranet as an example:

## Local(Outgoing)

    (http)
    Source host: localhost
    Listening port: xx
    Destination host: xx.xx.xx.xx
    Destination port: 80
    (novnc)
    Source host: localhost
    Listening port: xx
    Destination host: xx.xx.xx.xx
    Destination port : 6080

The browser does not need to set up a proxy when accessing, just need to enter `http://localhost:port` in the address bar.

## Dynamic(SOCKS4/5)

Listening port: xx

You need to set SOCKS4 or SOCKS5 proxy when accessing the browser, and you need to enter the url address of the intranet in the address bar.

General browser supports SOCKS4/5 proxy, also you can use some proxy tool such as the Proxy SwitchyOmega plug-in of chrome.

The two tunnel forwarding methods have their own characteristics. I always use the latter, because there are few settings. If you want to access other ports or other services, you only need to set up the jump server.

# Application Scenario 1
Use the XShell tunnel to connect to the intranet machine through the springboard machine. The springboard machine can be accessed from the public network or through the local area network, but the internal network node public network or local area network cannot be directly accessed.

In this case, access can be achieved by establishing a tunnel through Local (Outgoing) and Dynamic (SOCKS4/5) methods.

# Application Scenario 2
A company is connected to another company's network, or a laptop in a location needs to be connected to a computer in a closed network (a city's office location is connected to another city's closed data center), and it can be remotely transmitted through the Remote (Incoming) method Establish a tunnel to achieve access.
Of course, the second application scenario is often realized by using teamviewer software. Using teamviewer has many disadvantages, such as charging software, charging traffic, and occupying 100% of the screen resources of a computer. Use the Remote (Incoming) method to establish a tunnel, free of charge, save traffic, run in the background, and the computer can be used for other work. The disadvantage is that it is not as comprehensive as teamviewer. The connection established by the tunnel is mainly used for ssh purposes.

Remote incoming Remote (Incoming)
'Local' is used to map the service on the server to a certain port of the local machine, while 'Remote' is the opposite, it maps the local port to a certain port of the server, and accessing the port on the server is actually accessing the local port. machine service. It is similar to this: the port mapping mode of the firewall, and the difference is that the former is to open a hole in the company's firewall, which is a legal operation; the latter is to dig a hole, which is an illegal operation without the need for network administrators . The latter may bring the risk of leaks to the company, especially if the ssh port mapping is done, the entire internal network of the company can be accessed from the external server!

After the tunnel is established through Remote (Incoming), the computer of Company A can access the resources of Company B.

Specific steps:

1. Tunnel establishment
1) From the computer of company B, ssh to the cloud host or physical machine where the office computers of both companies are reachable
2) Configure the tunnel

2. 144.85.93 is a node in company B LAN

3. Access process
1) From the computer of company A, ssh to the cloud host or physical machine where the office computers of both companies are reachable
2) After successful login, execute ssh 127.0.0.1 -p 2345 to complete the login from the computer of company A The whole process of company B LA


# Application Scenario 3
data center. This scenario is similar to Scenario 2, except that Company B is replaced by a data center.
It's just that the conditions of the data center may not be as good as the company's office environment, and the network cannot be pulled casually. So at this time, you can use a notebook (the role of the notebook is equivalent to the office computer of company B in scenario 2). Use the dual network cards of the notebook, connect the wireless network card to the hotspot of the mobile phone, and connect the wired network card to the computer room of the data center. Other steps are exactly the same as Scenario 2. It can achieve the purpose of remotely connecting to the data center in other regions or other companies.

**Notebook dual network card configuration**


Need to modify the route of windows
route add 10.220.0.0 mask 255.255.0.0 192.168.1.1 -p
route delete 0.0.0.0 mask 0.0.0.0 192.168.1.1
The above two commands are to delete the default route through the wired network card, and the default route uses the wireless network card; Access 10.220.*.* (the network segment of the server in the data center) via a wired network card

In this way, the notebook can connect to the Internet and the data center at the same time, but during use, it is found that the default route through the wired network card will be automatically added after a period of time, resulting in the inability to access the Internet. Copy the following content into the newly created file, the suffix of the file name is bat, and then execute the bat file (used to periodically delete the default route of the wired network card)

    @echo off 

    route delete 10.220.0.0 mask 255.255.0.0
    route add 10.220.0.0 mask 255.255.0.0 192.168.1.1

    :start
    route delete 0.0.0.0 mask 0.0.0.0 192.168.1.1
    choice /t 5 /d y /n >nul

    goto start