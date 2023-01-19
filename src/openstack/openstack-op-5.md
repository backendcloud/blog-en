release time :2022-09-26 18:24

# windows2012 image uses virtio-SCSI disk driver blue screen problem
The blue screen is because the driver of the image at that time was the virtio driver, and the disk device required the scsi driver, so the image at that time would have a blue screen, and the image that was remade later, the disk driver was the scsi driver, and the image was uploaded. 2 parameters: hw_disk_bus=scsi, hw_scsi_model=virtio-scsi, there is no blue screen after the test

Pay attention to select the disk drive type when making an image and loading the scsi driver. Choose the scsi mode instead of the virtio mode.


# Nova is always dispatched to some fixed computing nodes, not evenly distributed to all computing nodes
Check nova filter and weight


# Virtual machine evacuation occurs during high availability of virtual machines, and normal nodes will also be evacuated
**Phenomenon:**

XXXX-HOST-YYYY34 node failure, high availability components should evacuate XXXX-HOST-YYYY34 node, actually not only XXXX-HOST-YYYY34 node was evacuated, XXXX-HOST-YYYY340 341 342349 and other nodes have been evacuated, 340349 are normal nodes, not faulty nodes.

**Cause Analysis:**

It should be a character match, as long as the node containing XXXX-HOST-YYYY34 is matched, it will be evacuated.

**Analysis code:**

High availability components, after monitoring a node failure, will call the nova api to execute the nova evacuation command.

**Relevant code:**

python-novaclient\novaclient\shell.py

```python
...
@utils.arg(
    '--strict',
    dest='strict',
    action='store_true',
    default=False,
    help=_('Evacuate host with exact hypervisor hostname match'))
def do_host_evacuate(cs, args):
    """Evacuate all instances from failed host."""
    response = []
    for server in _hyper_servers(cs, args.host, args.strict):
        response.append(_server_evacuate(cs, server, args))
    utils.print_list(response, [
        "Server UUID",
        "Evacuate Accepted",
        "Error Message",
    ])
```


> From the above code, it can be seen that the strict parameter defaults to false

```python
def _hyper_servers(cs, host, strict):
    hypervisors = cs.hypervisors.search(host, servers=True)
    for hyper in hypervisors:
        if strict and hyper.hypervisor_hostname != host:
            continue
        if hasattr(hyper, 'servers'):
            for server in hyper.servers:
                yield server
        if strict:
            break
    else:
        if strict:
            msg = (_("No hypervisor matching '%s' could be found.") %
                   host)
            raise exceptions.NotFound(404, msg)
```

> It can be seen from the above code that when the strict parameter is true, it is an exact match, otherwise it is an included match.

```python
def search(self, hypervisor_match, servers=False, detailed=False):
    """
    Get a list of matching hypervisors.

    :param hypervisor_match: The hypervisor host name or a portion of it.
        The hypervisor hosts are selected with the host name matching
        this pattern.
    :param servers: If True, server information is also retrieved.
    :param detailed: If True, detailed hypervisor information is returned.
        This requires API version 2.53 or greater.
    """
    # Starting with microversion 2.53, the /servers and /search routes are
    # deprecated and we get the same results using GET /os-hypervisors
    # using query parameters for the hostname pattern and servers.
    if self.api_version >= api_versions.APIVersion('2.53'):
        url = ('/os-hypervisors%s?hypervisor_hostname_pattern=%s' %
               ('/detail' if detailed else '',
                parse.quote(hypervisor_match, safe='')))
        if servers:
            url += '&with_servers=True'
    else:
        if detailed:
            raise exceptions.UnsupportedVersion(
                _('Parameter "detailed" requires API version 2.53 or '
                  'greater.'))
        target = 'servers' if servers else 'search'
        url = ('/os-hypervisors/%s/%s' %
               (parse.quote(hypervisor_match, safe=''), target))
    return self._list(url, 'hypervisors')
```

novaclient calls the nova rest api to query the matched hypervisor.

    GET
    /os-hypervisors/detail
    List Hypervisors Details

    GET
    /os-hypervisors/statistics
    Show Hypervisor Statistics (DEPRECATED)

    GET
    /os-hypervisors/{hypervisor_id}
    Show Hypervisor Details

    GET
    /os-hypervisors/{hypervisor_id}/uptime
    Show Hypervisor Uptime (DEPRECATED)

    GET
    /os-hypervisors/{hypervisor_hostname_pattern}/search
    Search Hypervisor (DEPRECATED)

    GET
    /os-hypervisors/{hypervisor_hostname_pattern}/servers
    List Hypervisor Servers (DEPRECATED)

The novaclient called by the high-availability component of the virtual machine does not add the strict parameter, so as long as the host name contains XXXX-HOST-YYYY34, the nodes will be evacuated.

Countermeasures:
1. Standardize the naming rules, use XXXX-HOST-YYYY034 within 1,000 computing nodes, and the number number is unified to 3 digits.
2. Modify the high-availability component code, and add the strict parameter when calling the novaclient code.

