release time :2022-04-24 11:00

The Pod of the Kubernetes cluster uses Openstack Cinder as the backend storage, and cinder-csi-plugin needs to be deployed

> https://github.com/kubernetes/cloud-provider-openstack

# Unable to connect to Keystone: Add custom domain name resolution for some Pods
Keystone url uses a domain name, you need to add domain name resolution in /etc/hosts

The following is just an example to illustrate how to add custom domain name resolution for Pod

![](2023-01-19-14-21-55.png)

If some Pods depend on specific domain name resolution, you can use the hostAliases provided by K8S to add hosts for some workloads if you do not want to configure dns resolution:

    spec:
    hostAliases:
    - hostnames: [ "harbor.example.com" ]
        ip: "10.10.10.10"


After adding, you can see that hosts has been added to /etc/hosts in the container:

    $ cat /etc/hosts
    ...
    # Entries added by HostAliases.
    10.10.10.10    harboar.example.com


# The metadata service is not up

    [developer@localhost ~]$ kubectl logs csi-cinder-nodeplugin-t8hcx -nkube-system -c cinder-csi-plugin
    I0424 11:10:56.531843       1 driver.go:73] Driver: cinder.csi.openstack.org
    I0424 11:10:56.531889       1 driver.go:74] Driver version: 1.3.2@latest
    I0424 11:10:56.531892       1 driver.go:75] CSI Spec version: 1.3.0
    I0424 11:10:56.531898       1 driver.go:104] Enabling controller service capability: LIST_VOLUMES
    I0424 11:10:56.531900       1 driver.go:104] Enabling controller service capability: CREATE_DELETE_VOLUME
    I0424 11:10:56.531907       1 driver.go:104] Enabling controller service capability: PUBLISH_UNPUBLISH_VOLUME
    I0424 11:10:56.531909       1 driver.go:104] Enabling controller service capability: CREATE_DELETE_SNAPSHOT
    I0424 11:10:56.531911       1 driver.go:104] Enabling controller service capability: LIST_SNAPSHOTS
    I0424 11:10:56.531913       1 driver.go:104] Enabling controller service capability: EXPAND_VOLUME
    I0424 11:10:56.531915       1 driver.go:104] Enabling controller service capability: CLONE_VOLUME
    I0424 11:10:56.531917       1 driver.go:104] Enabling controller service capability: LIST_VOLUMES_PUBLISHED_NODES
    I0424 11:10:56.531919       1 driver.go:116] Enabling volume access mode: SINGLE_NODE_WRITER
    I0424 11:10:56.531920       1 driver.go:126] Enabling node service capability: STAGE_UNSTAGE_VOLUME
    I0424 11:10:56.531923       1 driver.go:126] Enabling node service capability: EXPAND_VOLUME
    I0424 11:10:56.531928       1 driver.go:126] Enabling node service capability: GET_VOLUME_STATS
    I0424 11:10:56.532069       1 openstack.go:90] Block storage opts: {0 false false}
    I0424 11:10:57.181982       1 server.go:108] Listening for connections on address: &net.UnixAddr{Name:"/csi/csi.sock", Net:"unix"}
    E0424 11:10:58.771149       1 utils.go:85] GRPC error: rpc error: code = Internal desc = [NodeGetInfo] unable to retrieve instance id of node error fetching http://169.254.169.254/openstack/latest/meta_data.json: Get "http://169.254.169.254/openstack/latest/meta_data.json": dial tcp 169.254.169.254:80: connect: connection refused
    E0424 11:11:00.768432       1 utils.go:85] GRPC error: rpc error: code = Internal desc = [NodeGetInfo] unable to retrieve instance id of node error fetching http://169.254.169.254/openstack/latest/meta_data.json: Get "http://169.254.169.254/openstack/latest/meta_data.json": dial tcp 169.254.169.254:80: connect: connection refused
    E0424 11:11:13.787632       1 utils.go:85] GRPC error: rpc error: code = Internal desc = [NodeGetInfo] unable to retrieve instance id of node error fetching http://169.254.169.254/openstack/latest/meta_data.json: Get "http://169.254.169.254/openstack/latest/meta_data.json": dial tcp 169.254.169.254:80: connect: connection refused

> https://github.com/kubernetes/cloud-provider-openstack/issues/1127

# After devstack restarts, cinder reports an error when creating a volume
> https://www.codetd.com/en/article/13087314

# The cinder-csi-plugin node service is not up

    Normal   Provisioning          3m32s (x9 over 7m47s)  cinder.csi.openstack.org_csi-cinder-controllerplugin-667d467bf6-qfsnn_90ee191a-788a-4cda-82f2-eb61f37b20f5  External provisioner is provisioning volume for claim "default/csi-pvc-cinderplugin"
    Warning  ProvisioningFailed    3m32s (x9 over 7m47s)  cinder.csi.openstack.org_csi-cinder-controllerplugin-667d467bf6-qfsnn_90ee191a-788a-4cda-82f2-eb61f37b20f5  failed to provision volume with StorageClass "csi-sc-cinderplugin": error generating accessibility requirements: no available topology found
    Normal   ExternalProvisioning  97s (x26 over 7m47s)   persistentvolume-controller        waiting for a volume to be created, either by external provisioner "cinder.csi.openstack.org" or manually created by system administrator

> https://github.com/kubernetes-sigs/aws-ebs-csi-driver/issues/848
> https://github.com/hetznercloud/csi-driver/issues/92

# minikube - http: server gave HTTP response to HTTPS client
> https://minikube.sigs.k8s.io/docs/handbook/registry/

