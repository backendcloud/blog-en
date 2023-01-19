release time :2022-08-25 13:09


# download k3sup

    [root@centos9 ~]# curl -sLS https://get.k3sup.dev | sh
    x86_64
    Downloading package https://github.com/alexellis/k3sup/releases/download/0.12.0/k3sup as /tmp/k3sup
    Download complete.

    Running with sufficient permissions to attempt to move k3sup to /usr/local/bin
    New version of k3sup installed to /usr/local/bin
    _    _____                 
    | | _|___ / ___ _   _ _ __  
    | |/ / |_ \/ __| | | | '_ \ 
    |   < ___) \__ \ |_| | |_) |
    |_|\_\____/|___/\__,_| .__/ 
                        |_|    
    Version: 0.12.0
    Git Commit: c59d67b63ec76d5d5e399808cf4b11a1e02ddbc8

    Give your support to k3sup via GitHub Sponsors:

    https://github.com/sponsors/alexellis

    ================================================================
    Thanks for choosing k3sup.
    Support the project through GitHub Sponsors

    https://github.com/sponsors/alexellis
    ================================================================

    [root@centos9 ~]# ls
    anaconda-ks.cfg  cmss-gerrit  go  my-github  origin
    [root@centos9 ~]# sudo install k3sup /usr/local/bin/
    install: cannot stat 'k3sup': No such file or directory
    [root@centos9 ~]# k3sup 
    _    _____                 
    | | _|___ / ___ _   _ _ __  
    | |/ / |_ \/ __| | | | '_ \ 
    |   < ___) \__ \ |_| | |_) |
    |_|\_\____/|___/\__,_| .__/ 
                        |_|    
    Usage:
    k3sup [flags]
    k3sup [command]

    Available Commands:
    completion  Generate the autocompletion script for the specified shell
    help        Help about any command
    install     Install k3s on a server via SSH
    join        Install the k3s agent on a remote host and join it to an existing server
    update      Print update instructions
    version     Print the version

    Flags:
    -h, --help   help for k3sup

    Use "k3sup [command] --help" for more information about a command.


# Deploy k3s with k3sup

    [root@centos9 ~]# k3sup install --local
    Running: k3sup install
    2022/08/25 14:05:09 127.0.0.1
    Executing: curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC='server --tls-san 127.0.0.1' INSTALL_K3S_CHANNEL='stable' sh -

    [INFO]  Finding release for channel stable
    [INFO]  Using v1.24.3+k3s1 as release
    [INFO]  Downloading hash https://github.com/k3s-io/k3s/releases/download/v1.24.3+k3s1/sha256sum-amd64.txt
    [INFO]  Downloading binary https://github.com/k3s-io/k3s/releases/download/v1.24.3+k3s1/k3s
    [INFO]  Verifying binary download
    [INFO]  Installing k3s to /usr/local/bin/k3s
    Rancher K3s Common (stable)                     182  B/s | 2.0 kB     00:11    
    Last metadata expiration check: 0:00:04 ago on Thu 25 Aug 2022 02:05:30 PM CST.
    Dependencies resolved.
    ================================================================================
    Package         Arch       Version         Repository                     Size
    ================================================================================
    Installing:
    k3s-selinux     noarch     1.2-2.el8       rancher-k3s-common-stable      20 k

    Transaction Summary
    ================================================================================
    Install  1 Package

    Total download size: 20 k
    Installed size: 94 k
    Downloading Packages:
    k3s-selinux-1.2-2.el8.noarch.rpm                 23 kB/s |  20 kB     00:00    
    --------------------------------------------------------------------------------
    Total                                            23 kB/s |  20 kB     00:00     
    Rancher K3s Common (stable)                     4.9 kB/s | 2.4 kB     00:00    
    Importing GPG key 0xE257814A:
    Userid     : "Rancher (CI) <ci@rancher.com>"
    Fingerprint: C8CF F216 4551 26E9 B9C9 18BE 925E A29A E257 814A
    From       : https://rpm.rancher.io/public.key
    Key imported successfully
    Running transaction check
    Transaction check succeeded.
    Running transaction test
    Transaction test succeeded.
    Running transaction
    Preparing        :                                                        1/1 
    Running scriptlet: k3s-selinux-1.2-2.el8.noarch                           1/1 
    Installing       : k3s-selinux-1.2-2.el8.noarch                           1/1 
    Running scriptlet: k3s-selinux-1.2-2.el8.noarch                           1/1uavc:  op=load_policy lsm=selinux seqno=3 res=1 
    Verifying        : k3s-selinux-1.2-2.el8.noarch                           1/1 

    Installed:
    k3s-selinux-1.2-2.el8.noarch                                                  

    Complete!
    [INFO]  Skipping /usr/local/bin/kubectl symlink to k3s, command exists in PATH at /usr/bin/kubectl
    [INFO]  Creating /usr/local/bin/crictl symlink to k3s
    [INFO]  Creating /usr/local/bin/ctr symlink to k3s
    [INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
    [INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
    [INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
    [INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
    [INFO]  systemd: Enabling k3s unit
    Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
    [INFO]  systemd: Starting k3s
    stderr: "Importing GPG key 0xE257814A:\n Userid     : \"Rancher (CI) <ci@rancher.com>\"\n Fingerprint: C8CF F216 4551 26E9 B9C9 18BE 925E A29A E257 814A\n From       : https://rpm.rancher.io/public.key\nuavc:  op=load_policy lsm=selinux seqno=3 res=1Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.\n"stdout: "[INFO]  Finding release for channel stable\n[INFO]  Using v1.24.3+k3s1 as release\n[INFO]  Downloading hash https://github.com/k3s-io/k3s/releases/download/v1.24.3+k3s1/sha256sum-amd64.txt\n[INFO]  Downloading binary https://github.com/k3s-io/k3s/releases/download/v1.24.3+k3s1/k3s\n[INFO]  Verifying binary download\n[INFO]  Installing k3s to /usr/local/bin/k3s\nRancher K3s Common (stable)                     182  B/s | 2.0 kB     00:11    \nLast metadata expiration check: 0:00:04 ago on Thu 25 Aug 2022 02:05:30 PM CST.\nDependencies resolved.\n================================================================================\n Package         Arch       Version         Repository                     Size\n================================================================================\nInstalling:\n k3s-selinux     noarch     1.2-2.el8       rancher-k3s-common-stable      20 k\n\nTransaction Summary\n================================================================================\nInstall  1 Package\n\nTotal download size: 20 k\nInstalled size: 94 k\nDownloading Packages:\nk3s-selinux-1.2-2.el8.noarch.rpm                 23 kB/s |  20 kB     00:00    \n--------------------------------------------------------------------------------\nTotal                                            23 kB/s |  20 kB     00:00     \nRancher K3s Common (stable)                     4.9 kB/s | 2.4 kB     00:00    \nKey imported successfully\nRunning transaction check\nTransaction check succeeded.\nRunning transaction test\nTransaction test succeeded.\nRunning transaction\n  Preparing        :                                                        1/1 \n  Running scriptlet: k3s-selinux-1.2-2.el8.noarch                           1/1 \n  Installing       : k3s-selinux-1.2-2.el8.noarch                           1/1 \n  Running scriptlet: k3s-selinux-1.2-2.el8.noarch                           1/1 \n  Verifying        : k3s-selinux-1.2-2.el8.noarch                           1/1 \n\nInstalled:\n  k3s-selinux-1.2-2.el8.noarch                                                  \n\nComplete!\n[INFO]  Skipping /usr/local/bin/kubectl symlink to k3s, command exists in PATH at /usr/bin/kubectl\n[INFO]  Creating /usr/local/bin/crictl symlink to k3s\n[INFO]  Creating /usr/local/bin/ctr symlink to k3s\n[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh\n[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh\n[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env\n[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service\n[INFO]  systemd: Enabling k3s unit\n[INFO]  systemd: Starting k3s\n"Saving file to: /root/kubeconfig

    # Test your cluster with:
    export KUBECONFIG=/root/kubeconfig
    kubectl config set-context default
    kubectl get node -o wide
    [root@centos9 ~]# export KUBECONFIG=/root/kubeconfig
    [root@centos9 ~]# kubectl config set-context default
    Context "default" modified.
    [root@centos9 ~]# kubectl get node -o wide
    NAME      STATUS   ROLES                  AGE   VERSION        INTERNAL-IP       EXTERNAL-IP   OS-IMAGE          KERNEL-VERSION          CONTAINER-RUNTIME
    centos9   Ready    control-plane,master   38s   v1.24.3+k3s1   192.168.190.130   <none>        CentOS Stream 9   5.14.0-142.el9.x86_64   containerd://1.6.6-k3s1
    [root@centos9 ~]# kubectl get pod -A
    NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
    kube-system   local-path-provisioner-7b7dc8d6f5-2mzjb   1/1     Running     0          5m10s
    kube-system   coredns-b96499967-tdbw4                   1/1     Running     0          5m10s
    kube-system   metrics-server-668d979685-fhd8n           1/1     Running     0          5m10s
    kube-system   helm-install-traefik-crd-7s72z            0/1     Completed   0          5m10s
    kube-system   helm-install-traefik-c4r5v                0/1     Completed   0          5m10s
    kube-system   traefik-7cd4fcff68-97897                  0/1     Running     0          35s
    kube-system   svclb-traefik-66c6ef70-m6pgz              2/2     Running     0          35s



> The deployment of k3s is very simple, but it is faster to use the k3sup tool, especially the k3sup tool can specify the ip to remotely deploy to any remote server or cloud host.

> Deploying k8s and k3s, compared to creating a k8s cluster with kind, k3sup is much faster to create a k3s cluster. This speed is mainly because the k3s download file is much smaller, about one-tenth. But in fact, after the k3s cluster is ready, some helm install packages need to be executed, and k3s will be slower when the cluster is available.


# Comparison of k3s direct deployment and k3sup tool deployment commands
Comparison of deploying single-node server/agent commands

```bash
curl -sfL https://get.k3s.io | sh -
# Check for Ready node, takes maybe 30 seconds

kubectl get node  -o wide
kubectl get pod -A
```

```bash
k3sup install --ip $SERVER_IP --user $USER

kubectl get node  -o wide
kubectl get pod -A
```

Agent join server command comparison

```bash
sudo k3s server &
# Kubeconfig is written to /etc/rancher/k3s/k3s.yaml
sudo k3s kubectl get node
# On a different node run the below. NODE_TOKEN comes from /var/lib/rancher/k3s/server/node-token
# on your server
sudo k3s agent --server https://myserver:6443 --token ${NODE_TOKEN}

kubectl get node  -o wide
kubectl get pod -A
```

```bash
#Install the first server
k3sup install --user root --ip $SERVER1 --datastore="${DATASTORE}" --token=${TOKEN}
#Install the second server
k3sup install --user root --ip $SERVER2 --datastore="${DATASTORE}" --token=${TOKEN}
#Join the first agent
#You can join the agent to either server, the datastore is not required for this step.
k3sup join --user root --server-ip $SERVER1 --ip $AGENT1

kubectl get node  -o wide
kubectl get pod -A
```

# Uninstall K3s
If you installed K3s using the install script, a script to uninstall K3s is generated during the installation process.

To uninstall K3s from a server node, run:

/usr/local/bin/k3s-uninstall.sh

To uninstall K3s from an agent node, run:

/usr/local/bin/k3s-agent-uninstall.sh



# easy to use
## Create a pod that mounts local-path storage

    [root@centos9 tt]# cat pvc.yaml 
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
    name: local-path-pvc
    namespace: default
    spec:
    accessModes:
        - ReadWriteOnce
    storageClassName: local-path
    resources:
        requests:
        storage: 2Gi
    [root@centos9 tt]# cat pod.yaml 
    apiVersion: v1
    kind: Pod
    metadata:
    name: volume-test
    namespace: default
    spec:
    containers:
    - name: volume-test
        image: nginx:stable-alpine
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: volv
        mountPath: /data
        ports:
        - containerPort: 80
    volumes:
    - name: volv
        persistentVolumeClaim:
        claimName: local-path-pvc
    [root@centos9 tt]# kubectl create -f pvc.yaml
    persistentvolumeclaim/local-path-pvc created
    [root@centos9 tt]# kubectl create -f pod.yaml 
    pod/volume-test created
    [root@centos9 tt]# kubectl get pod -A
    NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
    kube-system   coredns-b96499967-x277m                   1/1     Running     0          2m6s
    kube-system   local-path-provisioner-7b7dc8d6f5-qxscd   1/1     Running     0          2m6s
    kube-system   metrics-server-668d979685-k7wl7           1/1     Running     0          2m6s
    kube-system   helm-install-traefik-crd-2lt5l            0/1     Completed   0          2m6s
    kube-system   helm-install-traefik-zm77d                0/1     Completed   1          2m6s
    kube-system   svclb-traefik-089ff14b-pf7b9              2/2     Running     0          56s
    kube-system   traefik-7cd4fcff68-ddr6q                  0/1     Running     0          56s
    default       volume-test                               1/1     Running     0          85s
    [root@centos9 tt]# kubectl get pod
    NAME          READY   STATUS    RESTARTS   AGE
    volume-test   1/1     Running   0          94s

## Create a pod that mounts longhorn storage

    [root@centos9 tt]# kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/master/deploy/longhorn.yaml
    namespace/longhorn-system created
    Warning: policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
    podsecuritypolicy.policy/longhorn-psp created
    serviceaccount/longhorn-service-account created
    configmap/longhorn-default-setting created
    configmap/longhorn-storageclass created
    customresourcedefinition.apiextensions.k8s.io/backingimagedatasources.longhorn.io created
    customresourcedefinition.apiextensions.k8s.io/backingimagemanagers.longhorn.io created
    customresourcedefinition.apiextensions.k8s.io/backingimages.longhorn.io created
    customresourcedefinition.apiextensions.k8s.io/backups.longhorn.io created
    customresourcedefinition.apiextensions.k8s.io/backuptargets.longhorn.io created
    customresourcedefinition.apiextensions.k8s.io/backupvolumes.longhorn.io created
    customresourcedefinition.apiextensions.k8s.io/engineimages.longhorn.io created
    customresourcedefinition.apiextensions.k8s.io/engines.longhorn.io created
    customresourcedefinition.apiextensions.k8s.io/instancemanagers.longhorn.io created
    customresourcedefinition.apiextensions.k8s.io/nodes.longhorn.io created
    customresourcedefinition.apiextensions.k8s.io/orphans.longhorn.io created
    customresourcedefinition.apiextensions.k8s.io/recurringjobs.longhorn.io created
    customresourcedefinition.apiextensions.k8s.io/replicas.longhorn.io created
    customresourcedefinition.apiextensions.k8s.io/settings.longhorn.io created
    customresourcedefinition.apiextensions.k8s.io/sharemanagers.longhorn.io created
    customresourcedefinition.apiextensions.k8s.io/snapshots.longhorn.io created
    customresourcedefinition.apiextensions.k8s.io/volumes.longhorn.io created
    clusterrole.rbac.authorization.k8s.io/longhorn-role created
    clusterrolebinding.rbac.authorization.k8s.io/longhorn-bind created
    role.rbac.authorization.k8s.io/longhorn-psp-role created
    rolebinding.rbac.authorization.k8s.io/longhorn-psp-binding created
    service/longhorn-backend created
    service/longhorn-frontend created
    service/longhorn-conversion-webhook created
    service/longhorn-admission-webhook created
    service/longhorn-engine-manager created
    service/longhorn-replica-manager created
    daemonset.apps/longhorn-manager created
    deployment.apps/longhorn-driver-deployer created
    deployment.apps/longhorn-ui created
    deployment.apps/longhorn-conversion-webhook created
    deployment.apps/longhorn-admission-webhook created
    [root@centos9 tt]# kubectl get pod -A
    NAMESPACE         NAME                                           READY   STATUS      RESTARTS   AGE
    kube-system       coredns-b96499967-x277m                        1/1     Running     0          38m
    kube-system       local-path-provisioner-7b7dc8d6f5-qxscd        1/1     Running     0          38m
    kube-system       metrics-server-668d979685-k7wl7                1/1     Running     0          38m
    kube-system       helm-install-traefik-crd-2lt5l                 0/1     Completed   0          38m
    kube-system       helm-install-traefik-zm77d                     0/1     Completed   1          38m
    kube-system       svclb-traefik-089ff14b-pf7b9                   2/2     Running     0          37m
    kube-system       traefik-7cd4fcff68-ddr6q                       1/1     Running     0          37m
    longhorn-system   longhorn-conversion-webhook-7b98d5c6cf-kjh9b   1/1     Running     0          33m
    longhorn-system   longhorn-conversion-webhook-7b98d5c6cf-g2q5v   1/1     Running     0          33m
    longhorn-system   longhorn-admission-webhook-6df4465d84-574qs    1/1     Running     0          33m
    longhorn-system   longhorn-admission-webhook-6df4465d84-tl6gh    1/1     Running     0          33m
    longhorn-system   longhorn-manager-j5n4z                         1/1     Running     0          33m
    longhorn-system   longhorn-driver-deployer-7cc677fb4f-zqxdg      1/1     Running     0          33m
    longhorn-system   csi-resizer-7c5bb5fd65-mx9b2                   1/1     Running     0          29m
    longhorn-system   csi-resizer-7c5bb5fd65-xxrpv                   1/1     Running     0          29m
    longhorn-system   csi-resizer-7c5bb5fd65-xknnl                   1/1     Running     0          29m
    longhorn-system   longhorn-csi-plugin-r4fqz                      2/2     Running     0          29m
    longhorn-system   csi-attacher-dcb85d774-w8j24                   1/1     Running     0          29m
    longhorn-system   csi-attacher-dcb85d774-9c4cc                   1/1     Running     0          29m
    longhorn-system   csi-attacher-dcb85d774-rnglx                   1/1     Running     0          29m
    longhorn-system   csi-provisioner-5d8dd96b57-zq57d               1/1     Running     0          29m
    longhorn-system   csi-provisioner-5d8dd96b57-dqdnf               1/1     Running     0          29m
    longhorn-system   csi-provisioner-5d8dd96b57-pkpwq               1/1     Running     0          29m
    longhorn-system   longhorn-ui-6bf766d6d6-jp79l                   1/1     Running     0          28m
    longhorn-system   csi-snapshotter-5586bc7c79-7zdzv               1/1     Running     0          29m
    longhorn-system   csi-snapshotter-5586bc7c79-g4w4b               1/1     Running     0          29m
    longhorn-system   csi-snapshotter-5586bc7c79-nfcd7               1/1     Running     0          29m
    longhorn-system   engine-image-ei-b907910b-flzfw                 1/1     Running     0          29m
    longhorn-system   instance-manager-r-a3c8b5e8                    1/1     Running     0          15m
    longhorn-system   instance-manager-e-615fda27                    1/1     Running     0          11m
    [root@centos9 tt]# cat pvc.yaml 
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
    name: longhorn-volv-pvc
    spec:
    accessModes:
        - ReadWriteOnce
    storageClassName: longhorn
    resources:
        requests:
        storage: 2Gi
    [root@centos9 tt]# cat pod.yaml 
    apiVersion: v1
    kind: Pod
    metadata:
    name: volume-test
    namespace: default
    spec:
    containers:
    - name: volume-test
        image: nginx:stable-alpine
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: volv
        mountPath: /data
        ports:
        - containerPort: 80
    volumes:
    - name: volv
        persistentVolumeClaim:
        claimName: longhorn-volv-pvc
    [root@centos9 tt]# kubectl create -f pvc.yaml 
    persistentvolumeclaim/longhorn-volv-pvc created
    [root@centos9 tt]# kubectl get pvc
    NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    longhorn-volv-pvc   Bound    pvc-7b491458-267d-4359-a078-5cc270a2bb5f   2Gi        RWO            longhorn       5s
    [root@centos9 tt]# kubectl get pv
    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                       STORAGECLASS   REASON   AGE
    pvc-7b491458-267d-4359-a078-5cc270a2bb5f   2Gi        RWO            Delete           Bound    default/longhorn-volv-pvc   longhorn                5s
    [root@centos9 tt]# kubectl create -f pod.yaml 
    pod/volume-test created
    [root@centos9 tt]# kubectl get pod
    NAME          READY   STATUS    RESTARTS   AGE
    volume-test   1/1     Running   0          23s



