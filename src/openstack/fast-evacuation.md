release time :2017-06-10 03:27

# Relevant blueprints:
* https://blueprints.launchpad.net/nova/+spec/mark-host-down
* https://blueprints.launchpad.net/python-novaclient/+spec/support-force-down-service

# Why add this API

With this API, external fault monitoring can get the nova-compute service down faster, which will directly improve the fault recovery time of VM HA (the time to evacuate a VM from a faulty node to a normal node).
external fault monitoring system a possibility of telling OpenStack Nova fast that compute host is down. This will immediately enable calling of evacuation of any VM on host and further enabling faster HA actions.

# REST API for forced down

request: PUT /v2.1/{tenant_id}/os-services/force-down { “binary”: “nova-compute”, “host”: “compute1”, “forced_down”: true }

response: 200 OK { “service”: { “host”: “compute1”, “binary”: “nova-compute”, “forced_down”: true } }

Example: curl -g -i -X PUT http://{service_host_ip}:8774/v2.1/{tenant_id}/os-services /force-down -H “Content-Type: application/json” -H “Accept: application/json ” -H “X-OpenStack-Nova-API-Version: 2.11” -H “X-Auth-Token: {token}” -d ‘{“b inary”: “nova-compute”, “host”: “compute1”, “forced_down”: true}’

# CLI for forced down

nova service-force-down nova-compute

Example: nova service-force-down compute1 nova-compute

# REST API for disabling forced down

request: PUT /v2.1/{tenant_id}/os-services/force-down { “binary”: “nova-compute”, “host”: “compute1”, “forced_down”: false }

response: 200 OK { “service”: { “host”: “compute1”, “binary”: “nova-compute”, “forced_down”: false } }

Example: curl -g -i -X PUT http://{service_host_ip}:8774/v2.1/{tenant_id}/os-services /force-down -H “Content-Type: application/json” -H “Accept: application/json ” -H “X-OpenStack-Nova-API-Version: 2.11” -H “X-Auth-Token: {token}” -d ‘{“b inary”: “nova-compute”, “host”: “compute1”, “forced_down”: false}’

# CLI for disabling forced down

nova service-force-down –unset nova-compute

Example: nova service-force-down –unset compute1 nova-compute


# Add fast evacuation function to VM HA procedure

Now add fast evacuation function in the developed VM HA program

1. Modify the configuration file and add the quick_evacuate_level option
* level 1: (fastest) Do not shut down, just force down nova-compute-service
* level 2: (faster) (default) Shut down first, then force down nova-compute-service
* level 3: (disable quick_evacuate)
2. change python-novaclient>=6.0.0 to support force-service-down
3. Modify the processing logic of the evacuation process and add rapid evacuation

The original evacuation process is to shut down the faulty computing node after finding the faulty node, and then wait for the faulty node to cause nova service state down due to heartbeat loss, and then perform evacuation.

Now it is changed to:

* level 1: (fastest) does not shut down, only force down nova-compute -service
* level 2: (faster) (default) Shut down first, then force down after shutdown introduction nova-compute-service
* level 3: (disable quick_evacuate) do nothing, same as the original process

The following code def _ipmi_handle, before the change, shuts down the faulty node. After the change, at level 1, force down nova service state and skip the shutdown process.

```python
def _ipmi_handle(self, node):
    # cmd = 'ssh %s systemctl stop openstack-nova-compute' % node
    # return os.system(cmd)
    ret = 1
################  add  ################
    if CONF.quick_evacuate_level == 1:
        try:
            self.nova_client.services.force_down(host=node,
                                                 binary='nova-compute',
                                                 force_down=True,
                                                )
            service = self.nova_client.services.list(
                host=node,
                binary='nova-compute')
            if service:
                if service[0].state == 'down':
                    return ret
            raise
        except Exception as e:
            content = node + ' nova-compute service force down failed!'
            title = 'force down failed'
            self.db.add_or_update_guardian_log(**{'title': title,
                                                  'detail': content,
                                                  'level': 'ERROR'})
            LOG.error(content)
            LOG.error(e)
################  add  ################
    LOG.info('IPMI enabled,need to check power status of %s' % node)
    ipmi_info = self.db.get_ipmiInfo(node)
    if ipmi_info is None:
        LOG.warning(
            'can not find ipmi info for %s, will ignore evacuate...'
            % node)
        return 0
    LOG.debug('get ipmi info (ip %s user %s, password %s)' % (
        ipmi_info.ip, ipmi_info.username, ipmi_info.password))
    ipmi_util = IPMIUtil(ipmi_info.ip, ipmi_info.username,
                         ipmi_info.password)
    power_status = ipmi_util.get_power_status()
    LOG.info('power status of %s is %s' % (node, power_status))

    if 'off' != power_status:
        LOG.info('power status of %s is not off,power off it...' % node)
        ipmi_util.do_power_off()
        for i in range(1, CONF.ipmi_check_max_count + 1):
            power_status = ipmi_util.get_power_status()
            if 'off' != power_status:
                LOG.info(
                    'power status of %s is not off,wait for %s seconds...'
                    % (node, CONF.ipmi_check_interval))
                time.sleep(CONF.ipmi_check_interval)
            else:
                break
        power_status = ipmi_util.get_power_status()
        if 'off' != power_status:
            LOG.info(
                'after %s check,can not confirm power status of '
                '%s is off,will ignore evacuate...'
                % (node,
                   CONF.ipmi_check_max_count*CONF.ipmi_check_interval))
            ret = 0
    return ret
```

When the following code is at level 2, it does not skip the shutdown process of the above code, and force down before checking the nova service state to achieve rapid evacuation (do not check the heartbeat process). Level 3 is to do nothing, the same as before.

```python
################  add  ################
if CONF.quick_evacuate_level == 2:
    try:
        self.nova_client.services.force_down(host=node,
                                             binary='nova-compute',
                                             force_down=True,
                                            )
        service = self.nova_client.services.list(
            host=node,
            binary='nova-compute')
        if service:
            if service[0].state != 'down':
                raise
    except Exception as e:
        content = node + ' nova-compute service force down failed!'
        title = 'force down failed'
        self.db.add_or_update_guardian_log(**{'title': title,
                                              'detail': content,
                                              'level': 'ERROR'})
        LOG.error(content)
        LOG.error(e)
################  add  ################

if self._wait_for_service_to_down(node):
    for server in servers:
        can_evacuate = self._evacuate_predict(server)
        if can_evacuate:
            try:
                # just evacuate, let nova scheduler
                # choose one host if not in drbd env
                respone = server.evacuate(
                    host=evacuated_to,
                    on_shared_storage=CONF.on_share_storage)
                LOG.info(
                    'evacuate vm info:[id:%s,name:%s,status:%s],'
                    'reponse is %s'
                    % (server.id, server.name, server.status,
                        respone))
                content = '虚拟机被疏散:虚拟机ID: %s' % server.id
                self.send_snmp(content=content)
            except Exception as e:
                content = ('evacuate vm %s failed' % server.id)
                title = ('疏散虚拟机%s失败' % server.id)
                self.db.add_or_update_guardian_log(
                    **{'title': title,
                       'detail': content,
                       'level': 'ERROR'})
                content = '虚拟机疏散调用API失败:虚拟机ID: %s' % server.id
                self.send_snmp(content=content)
                LOG.error(content)
                LOG.error(e)

            # add to db
            try:
                self.db.add_evacuate_history(
                    instance_id=server.id,
                    instance_name=server.name,
                    host=getattr(server, 'OS-EXT-SRV-ATTR:host'))

                content = (
                    'add_evacuate_history '
                    '[instance_id:%s,instance_name:%s,host:%s]'
                    % (server.id, server.name,
                       getattr(server, 'OS-EXT-SRV-ATTR:host')))
                LOG.info(content)
                title = ('增加疏散记录%s' % server.name)
                self.db.add_or_update_guardian_log(
                    **{'title': title,
                       'detail': content,
                       'level': 'INFO'})
            except Exception as e:
                content = (
                    'add_evacuate_history for vm %s failed'
                    % server.id)
                title = ('增加疏散记录%s失败' % server.id)
                self.db.add_or_update_guardian_log(
                    **{'title': title,
                       'detail': content,
                       'level': 'ERROR'})
                LOG.error(content)
                LOG.error(e)
        else:
            content = '虚拟机没有被疏散:虚拟机ID: %s' % server.id
            self.send_snmp(content=content)
            LOG.info('server %s can not evacuate' % server.id)

    # disable this service
    try:
        self.nova_client.services.disable(node, 'nova-compute')
    except Exception as e:
        content = (
            'failed to disable compute service on node %s' % node)
        title = ('disable节点%s服务失败' % node)
        self.db.add_or_update_guardian_log(
            **{'title': title,
               'detail': content,
               'level': 'ERROR'})
        LOG.error(content)
        LOG.error(e)
        raise

    heart_beat = self.db.get_heartbeat_by_name(node)
    heart_beat.has_evacuated = 1
    heart_beat.status = 'done'
    heart_beat.save(self.db.session)
    LOG.info('node %s update with [has_evacuated: 1]' % node)
```