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