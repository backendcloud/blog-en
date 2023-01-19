release time :2020-05-22 13:52

Run the increased server memory stress test script

    #!/bin/bash
    mkdir /tmp/skyfans/memory -p
    mount -t tmpfs -o size=10240M tmpfs /tmp/skyfans/memory
    dd if=/dev/zero of=/tmp/skyfans/memory/block
    sleep 60
    rm /tmp/skyfans/memory/block
    umount /tmp/skyfans/memory
    rmdir /tmp/skyfans/memory

It is available on the vm and the server, but an error is reported when the mount is executed in the container: mount: permission denied
needs to enable privileged.
Privileged was introduced into docker around version 0.6.
With this parameter, the root inside the container has real root authority.
Otherwise, the root inside the container is just an ordinary user outside the authority.
The container started by privileged can see many devices on the host and can execute mount.
Even allows launching docker containers inside docker containers.

docker solution: add –privileged when docker run starts the container, such as: docker run -v /home/liurizhou/temp:/home/liurizhou/temp –privileged my_images:latest /bin.bash

k8s solution: add in containers: on securityContext: privileged: true runAsUser: 0


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mem-nginx
spec:
  containers:
  - name: mem-nginx-container
    image: registry.paas/library/nginx:1.15.9
    resources:
      requests:
        memory: "200Mi"
        cpu: "2"
      limits:
        memory: "200Mi"
        cpu: "2"
    volumeMounts:
      - mountPath: /etc/work
        name: host5490
    securityContext:
      privileged: true
      runAsUser: 0
  volumes:
  - hostPath:
        path: /docker_temp
    name: host5490
  nodeSelector:
    disktype: abc
```

Then execute the memory pressurization script, and mount can be executed.
As long as the node has enough memory resources, the container can use more memory than it requested, but the container is not allowed to use resources beyond its limit. If a container allocates more memory than the limit, the container will be terminated first. If the container continues to use more memory than the limit, the container will be terminated. If a terminated container is allowed to be restarted, the kubelet will restart the container.
For example, the upper limit in the above yaml file is 200M. After the memory pressure exceeds 200M, the pod will trigger OOMKilled to be terminated, and a new pod will be created.

    [root@paasm1 ~]# kubectl describe pod mem-nginx
    Name:               mem-nginx
    Namespace:          default
    Priority:           0
    PriorityClassName:  <none>
    Node:               paasn3/39.135.102.123
    Start Time:         Thu, 21 May 2020 18:08:50 +0800
    Labels:             <none>
    Annotations:        <none>
    Status:             Running
    IP:                 10.222.72.25
    Containers:
    mem-nginx-container:
        Container ID:   docker://e103444b362d34076290839d0c1ab593e2a79f17d8b373c787f1cb5ab2960942
        Image:          registry.paas/library/nginx:1.15.9
        Image ID:       docker-pullable://registry.paas/library/nginx@sha256:082b7224e857035c61eb4a802bfccf8392736953f78a99096acc7e3296739889
        Port:           <none>
        Host Port:      <none>
        State:          Running
        Started:      Fri, 22 May 2020 11:20:13 +0800
        Last State:     Terminated
        Reason:       OOMKilled
        Exit Code:    0
        Started:      Fri, 22 May 2020 11:12:10 +0800
        Finished:     Fri, 22 May 2020 11:20:11 +0800
        Ready:          True
        Restart Count:  2
        Limits:
        cpu:     2
        memory:  200Mi
        Requests:
        cpu:        2
        memory:     200Mi
        Environment:  <none>
        Mounts:
        /etc/work from host5490 (rw)
        /var/run/secrets/kubernetes.io/serviceaccount from default-token-vc2bx (ro)
    Conditions:
    Type              Status
    Initialized       True
    Ready             True
    ContainersReady   True
    PodScheduled      True
    Volumes:
    host5490:
        Type:          HostPath (bare host directory volume)
        Path:          /docker_temp
        HostPathType:
    default-token-vc2bx:
        Type:        Secret (a volume populated by a Secret)
        SecretName:  default-token-vc2bx
        Optional:    false
    QoS Class:       Guaranteed
    Node-Selectors:  disktype=abc
    Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                    node.kubernetes.io/unreachable:NoExecute for 300s
    Events:
    Type    Reason          Age                 From             Message
    ----    ------          ----                ----             -------
    Normal  SandboxChanged  2m3s (x2 over 10m)  kubelet, paasn3  Pod sandbox changed, it will be killed and re-created.
    Normal  Killing         2m2s (x2 over 10m)  kubelet, paasn3  Killing container with id docker://mem-nginx-container:Need to kill Pod
    Normal  Pulled          2m (x3 over 17h)    kubelet, paasn3  Container image "registry.paas/library/nginx:1.15.9" already present on machine
    Normal  Created         2m (x3 over 17h)    kubelet, paasn3  Created container
    Normal  Started         2m (x3 over 17h)    kubelet, paasn3  Started container

docker ps to view the container and found that k8s_mem-nginx-container_mem-nginx_default_116a11b8-9b4b-11ea-a000-04bd7053eff0_1it has becomek8s_mem-nginx-container_mem-nginx_default_116a11b8-9b4b-11ea-a000-04bd7053eff0_2
