release time :2022-09-13 18:14


# Problem: Old applications cannot be accessed through kubectl exec, and new applications cannot be created
Ssh into the cluster node, telnet the local kubelete service port 10250, yes.

telnet port 10250 of other nodes from this machine fails.

The cluster has made a network policy. The 10250 port policy needs to be released.

In the test environment, if the node where the cluster is located is a vm, you can turn off port-security through Openstack: neutron port-update port-id-zzzzz –port-security_enabled=False

# Issue: Modify pod-cidr-range (CNI: calico)
1. Install calicoctl as a Kubernetes pod

    # kubectl apply -f https://docs.projectcalico.org/manifests/calicoctl.yaml
    # alias calicoctl="kubectl exec -i -n kube-system calicoctl -- /calicoctl "

2. Add an IP pool

    calicoctl create -f -<<EOF
    apiVersion: projectcalico.org/v3
    kind: IPPool
    metadata:
    name: new-pool
    spec:
    cidr: 10.0.0.0/8
    ipipMode: Always
    natOutgoing: true
    EOF

calicoctl get ippool -o wide can see one more line

3. Disable old IP pool

calicoctl get ippool -o yaml > pool.yaml

vim pool.yaml , modify or add:disabled: true

    apiVersion: projectcalico.org/v3
    kind: IPPool
    metadata:
    name: default-ipv4-ippool
    spec:
    cidr: 192.168.0.0/16
    ipipMode: Always
    natOutgoing: true
    disabled: true

calicoctl apply -f pool.yaml

4. Modify the podCIDR of all nodes

    $ kubectl get no kubeadm-0 -o yaml > file.yaml; sed -i "s~192.168.0.0/24~10.0.0.0/16~" file.yaml; kubectl delete no kubeadm-0 && kubectl create -f file.yaml
    $ kubectl get no kubeadm-1 -o yaml > file.yaml; sed -i "s~192.168.1.0/24~10.1.0.0/16~" file.yaml; kubectl delete no kubeadm-1 && kubectl create -f file.yaml
    $ kubectl get no kubeadm-2 -o yaml > file.yaml; sed -i "s~192.168.2.0/24~10.2.0.0/16~" file.yaml; kubectl delete no kubeadm-2 && kubectl create -f file.yaml 


5. Modify kubeadm-config and kube-proxy ConfigMap and kube-controller-manager.yaml

kubectl -n kube-system edit cm kubeadm-config

kubectl -n kube-system edit cm kube-proxy

vim /etc/kubernetes/manifests/kube-controller-manager.yaml

Modify the content of –cluster-cidr=xxx


6. Delete existing workloads using IPs from the disabled pool. (For example: during cluster deployment, the first coredns to use the calico ip pool after the calico service is normal)

kubectl delete pod -n kube-system coredns-5495dd7c88-b2zcs

Use calicoctl get wep --all-namespaces to check whether the newly generated coredns pod uses the new calico ip pool ip.

7. Delete old IP pool

calicoctl delete pool default-ipv4-ippool

# Problem: Failed to mount API filesystems, freezing.

    Failed to mount tmpfs at /run: Operation not permitted
    Failed to mount cgroup at /sys/fs/cgroup/systemd: Operation not permitted
    [!!!!!!] Failed to mount API filesystems, freezing.
    # 解决方法：挂载宿主机的cgroup
    docker run -it -d --name=centos7 --privileged=true -p 80:80 -v /sys/fs/cgroup:/sys/fs/cgroup:ro sevming/centos7:0.1
    docker exec -it centos7 /bin/bash
    systemctl status

> If you run systemd in a container, the above method is not a good way, it is recommended to use the official image: https://hub.docker.com/r/centos/systemd

# Problem: ingress exposes port 30067 through hostport and cannot be accessed
The inspection found that the port 80 used by the ingress controller conflicts with haproxy

# Problem: Pods on different nodes cannot communicate
/proc/sys/net/ipv4/ip_forward is 0, and the ip forwarding function is disabled, resulting in the inability to access the pod. Change it to 1 to solve the problem.

# Problem: The service port can be accessed on the node where the pod is located, but the service port cannot be accessed in the pod
If the route in the pod is lost, restart the CNI network plug-in to restore the pod routing information.

# Problem: The v6 address of the cluster node cannot be accessed in the container
calicoctl edit ippool default-ipv6-ippool

Add parameter natOutgoing: true

# Istio ipv4 v6 dual-stack deployment is ok in some environments, but has problems in some environments (from the client curl server)
By grabbing the 15001 port of the sidecar, the tcp handshake on port 15001 in the problematic environment will fail, and there will be no ack response, but the tcp handshake will succeed in the ok environment, and the request will be processed normally.

The kernel version of the environment in question does not support iptables forwarding for ipv6. ok environment kernel support. So it can be solved by upgrading the kernel version.

Also try similar rules on the host, such as:

ip6tables -t nat -I OUTPUT -p tcp –dport 30022 -j REDIRECT –to-ports 22

then verify

ssh fd15:4ba5:5a2b:1008:20c:29ff:fe7c:bcd1 -p 30022

If the forwarding is correct, you can ssh, otherwise you can't.

> istio uses iptables to direct all outgoing traffic to the local 15001, and can also be tested on the host through similar configurations. All outgoing traffic to 30022 is directed to the local 22. The principle is the same as istio.


# Issue: Certificate handling

    # Check the certificate validity period
    [root@kubevirt ~]# kubeadm certs check-expiration
    [check-expiration] Reading configuration from the cluster...
    [check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
    W0914 16:43:35.868627   31135 utils.go:69] The recommended value for "clusterDNS" in "KubeletConfiguration" is: [10.233.0.10]; the provided value is: [169.254.25.10]

    CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
    admin.conf                 May 25, 2023 09:01 UTC   253d                                    no      
    apiserver                  May 25, 2023 09:01 UTC   253d            ca                      no      
    apiserver-kubelet-client   May 25, 2023 09:01 UTC   253d            ca                      no      
    controller-manager.conf    May 25, 2023 09:01 UTC   253d                                    no      
    front-proxy-client         May 25, 2023 09:01 UTC   253d            front-proxy-ca          no      
    scheduler.conf             May 25, 2023 09:01 UTC   253d                                    no      

    CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
    ca                      May 22, 2032 09:01 UTC   9y              no      
    front-proxy-ca          May 22, 2032 09:01 UTC   9y              no      
    # Check whether the certificate is automatically renewed
    [root@kubevirt ~]# kubectl get cm -o yaml -n kube-system kubeadm-config | grep RotateKubeletServerCertificate
            feature-gates: RotateKubeletServerCertificate=true,TTLAfterFinished=true,ExpandCSIVolumes=true,CSIStorageCapacity=true
            feature-gates: RotateKubeletServerCertificate=true,TTLAfterFinished=true,ExpandCSIVolumes=true,CSIStorageCapacity=true
            feature-gates: RotateKubeletServerCertificate=true,TTLAfterFinished=true,ExpandCSIVolumes=true,CSIStorageCapacity=true
    [root@kubevirt ~]# kubectl get no
    NAME       STATUS   ROLES                         AGE    VERSION
    kubevirt   Ready    control-plane,master,worker   111d   v1.21.5
    # Update the validity period of the certificate (the method below is only applicable to Kubernetes 1.21 and above)
    [root@kubevirt ~]# kubeadm certs renew all
    [renew] Reading configuration from the cluster...
    [renew] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
    W0914 16:45:30.798557   46420 utils.go:69] The recommended value for "clusterDNS" in "KubeletConfiguration" is: [10.233.0.10]; the provided value is: [169.254.25.10]

    certificate embedded in the kubeconfig file for the admin to use and for kubeadm itself renewed
    certificate for serving the Kubernetes API renewed
    certificate for the API server to connect to kubelet renewed
    certificate embedded in the kubeconfig file for the controller manager to use renewed
    certificate for the front proxy client renewed
    certificate embedded in the kubeconfig file for the scheduler manager to use renewed

    Done renewing certificates. You must restart the kube-apiserver, kube-controller-manager, kube-scheduler and etcd, so that they can use the new certificates.
    [root@kubevirt ~]# kubeadm certs check-expiration
    [check-expiration] Reading configuration from the cluster...
    [check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
    W0914 16:45:44.191471   48482 utils.go:69] The recommended value for "clusterDNS" in "KubeletConfiguration" is: [10.233.0.10]; the provided value is: [169.254.25.10]

    CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
    admin.conf                 Sep 14, 2023 08:45 UTC   364d                                    no      
    apiserver                  Sep 14, 2023 08:45 UTC   364d            ca                      no      
    apiserver-kubelet-client   Sep 14, 2023 08:45 UTC   364d            ca                      no      
    controller-manager.conf    Sep 14, 2023 08:45 UTC   364d                                    no      
    front-proxy-client         Sep 14, 2023 08:45 UTC   364d            front-proxy-ca          no      
    scheduler.conf             Sep 14, 2023 08:45 UTC   364d                                    no      

    CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
    ca                      May 22, 2032 09:01 UTC   9y              no      
    front-proxy-ca          May 22, 2032 09:01 UTC   9y              no  

    [root@kubevirt manifests]# pwd
    /etc/kubernetes/manifests
    [root@kubevirt manifests]# grep RotateKubeletServerCertificate *
    kube-apiserver.yaml:    - --feature-gates=RotateKubeletServerCertificate=true,TTLAfterFinished=true,ExpandCSIVolumes=true,CSIStorageCapacity=true
    kube-controller-manager.yaml:    - --feature-gates=RotateKubeletServerCertificate=true,TTLAfterFinished=true,ExpandCSIVolumes=true,CSIStorageCapacity=true
    kube-scheduler.yaml:    - --feature-gates=RotateKubeletServerCertificate=true,TTLAfterFinished=true,ExpandCSIVolumes=true,CSIStorageCapacity=true


# Problem: [emerg] bind() to 0.0.0.0:801 failed (13: Permission denied)
kubectl logs -f pod-xxx reports an error:

[emerg] bind() to 0.0.0.0:801 failed (13: Permission denied)

Check the abnormal pod log, indicating insufficient permissions.

Entering the pod discovery is started by a non-root user:

    [root@centos7 ~]# kubectl exec  ingress-nginx-controller-7b768967bc-fd2hg -ningress-nginx -- id
    uid=101(www-data) gid=82(www-data)

sysctl -a|grep unprivileged_portNo information was found, indicating that the value of this parameter is the default value of 1024 (the process number from 1 to 1023 is reserved for the root user, and the default value is 1024), so port 801 reported insufficient permissions.

2 countermeasures (choose 1 from 2):

1. sysctl -w net.ipv4.ip_unprivileged_port_start=400(Through sysctl -w, the parameters can only take effect immediately (temporary effect). If you want to take effect permanently, you need to configure the parameters to /etc/sysctl.conf)
2. Containers use port 1024 or greater
