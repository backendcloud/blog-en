release time :2022-05-07 09:41

I took a small open source project on the Internet and changed it. The original project was written by vue on the front end and nodejs on the back end. The implemented function is a small tool for the firewall, which can display the background firewall policy through the front-end web interface, and can also insert a firewall policy through the interface. The front and back end use websocket to communicate.

The original project is an iptables graphical manager with simple addition, deletion, modification and query. Has the following functions:
* Specify table specify chain add rule
* Specified table Specified chain Specified position Insertion rule
* Clear the specified table specified chain rule
* View the referenced custom chain rule separately, which is more clear at a glance (CTRL key + left mouse button)

I made a little extension and changed a few lines of code. The extended functions implemented include:
* The original project can only operate on the firewall of the node deployed by nodejs, and now it can be extended to operate on the firewall of other nodes
* Added getting SSH security logs of other nodes
* Added the ability to obtain system logs of other nodes SystemLog

Therefore, the front-end and back-end communication formats have changed a little, adding a node field and adding two types, SystemLog and SSHLog

    {node: “10.253.199.82”, type: “SystemLog”}
    {node: “10.253.199.82”, type: “SSHLog”}

> Source code: https://github.com/backendcloud/iptables-UI