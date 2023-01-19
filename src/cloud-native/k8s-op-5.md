release time :2023-01-11 13:01


# Reasonably set Request and Limit
All containers should set request

The value of request does not refer to the actual resource size allocated to the container, it is only for the scheduler to see, the scheduler will "observe" how many resources each node can allocate, and know that each node has been allocated How many resources. The size of the allocated resources is the sum of the container requests defined in all Pods on the node, which can calculate how much resources the node has left that can be allocated (allocated resources minus the sum of allocated requests). If it is found that the size of the remaining allocatable resources of the node is smaller than the reuqest of the currently scheduled Pod, then this node will not be considered for scheduling, otherwise, scheduling is possible. Therefore, if the request is not configured, the scheduler cannot know how many resources are allocated to the node, and the scheduler cannot obtain accurate information, so it cannot make reasonable scheduling decisions, which can easily cause unreasonable scheduling, and some nodes may Very idle, while some nodes may be very busy, even NotReady.

When the node resources are insufficient, automatic eviction will be triggered, and some low-priority Pods will be deleted to release resources and allow the node to heal itself. Pods without request and limit have the lowest priority and are easy to be evicted; those with request not equal to limit have the second priority; Pods with request equal to limit have higher priority and are not easy to be evicted. Therefore, if it is an important online application and you do not want to be evicted when the node fails and the online business will be affected, it is recommended to set the request and limit to be consistent.

Therefore, the suggestion is to set a request for all containers, so that the scheduler can perceive how many resources are allocated to the nodes, so as to make reasonable scheduling decisions, so that the resources of the cluster nodes can be allocated and used reasonably, and avoid falling into the situation of uneven resource allocation. Some accidents happen.

Sometimes we forget to set request and limit for some containers. In fact, we can use LimitRange to set the default request and limit value of namespace, and it can also be used to limit the minimum and maximum request and limit. Example:

    apiVersion: v1
    kind: LimitRange
    metadata:
    name: mem-limit-range
    namespace: test
    spec:
    limits:
    - default:
        memory: 512Mi
        cpu: 500m
        defaultRequest:
        memory: 256Mi
        cpu: 100m
        type: Container


Try to avoid using too large request and limit

If your service uses a single copy or a small number of copies, give a large request and limit, and let it allocate enough resources to support the business, then a copy failure may have a greater impact on the business, and due to the request Larger, when the resource allocation in the cluster is relatively fragmented, if the node where the Pod is located hangs up, and none of the other nodes has enough remaining allocable resources to meet the request of the Pod, the Pod cannot drift, and it cannot Self-healing, aggravating the impact on the business.

On the contrary, it is recommended to reduce the request and limit as much as possible, and expand your service support capacity horizontally by adding copies to make your system more flexible and reliable.

If the production cluster has a namespace for testing, if it is not restricted, the cluster load may be too high, thereby affecting the production business. You can use ResourceQuota to limit the total size of the request and limit of the test namespace. Example:

    apiVersion: v1
    kind: ResourceQuota
    metadata:
    name: quota-test
    namespace: test
    spec:
    hard:
        requests.cpu: "1"
        requests.memory: 1Gi
        limits.cpu: "2"
        limits.memory: 2Gi


# pod cpu binding core and binding numa affinity

    # Evict nodes
    kubectl drain <NODE_NAME>
    # stop kubelet
    systemctl stop kubelet
    # Modify kubelet parameters
    # If you only need pod cpu binding core, just configure the following parameter
    --cpu-manager-policy=static
    # If you bind numa affinity, you need to configure the following two parameters
    --cpu-manager-policy=static
    --topology-manager-policy=single-numa-node
    # delete old CPU manager state files
    rm var/lib/kubelet/cpu_manager_state
    # start kubelet
    systemctl start kubelet

When creating a pod, the cpu request is strictly consistent with the limit.

Verification can be done by creating a process in the pod, so that all the CPUs of the pod can be used at 100%. Then log in to the node where the pod is located, and check it with cpustats. The CPUs with 100% CPU load are distributed on the same numa.

If you continue to create a pod, even if the CPU resources are sufficient, if the numa resources are not enough (that is, no numa can meet the number of CPUs required by the pod), the pod cannot be started, and the status will change to TopologyAffinityError.

# For high concurrency scenarios, expand the range of source ports

In high-concurrency scenarios, a large number of source ports will be used for the client. The source port range is randomly selected from the interval defined in the kernel parameter net.ipv4.ip_local_port_range. In a high-concurrency environment, a small port range will easily lead to exhaustion of source ports, making Some connections are abnormal. Usually, the Pod source port range is 32768-60999 by default. It is recommended to expand it to 1024-65535: sysctl -w net.ipv4.ip_local_port_range="1024 65535".

Two versions:
1. Most articles say that this value determines the number of ports available for one IP of the client, that is, one IP can only create a little more than 60K connections (1025-65535), if you want to break through this limit, you need to bind multiple IPs to the client machine .
2. Some articles also said that this value determines the number of local ports in the socket quadruple, that is, one ip can create a maximum of 60K connections to the same target ip+port, as long as the target ip or port is different, it can be used The same local port does not necessarily require multiple client ips to break through the limit on the number of ports.


Verification experiment:

First set the value of ip_local_port_range to a very small range:

    $ echo "61000 61001" | sudo tee /proc/sys/net/ipv4/ip_local_port_range
    61000 61001

    $ cat /proc/sys/net/ipv4/ip_local_port_range
    61000       61001

*Experiment 1: Limit the number of ports under the same target ip and same target port*

    $ nohup nc 123.125.114.144 80 -v &
    [1] 16196
    $ nohup: ignoring input and appending output to 'nohup.out'

    $ nohup nc 123.125.114.144 80 -v &
    [2] 16197
    $ nohup: ignoring input and appending output to 'nohup.out'

    $ ss -ant |grep 10.0.2.15:61
    ESTAB   0        0                10.0.2.15:61001       123.125.114.144:80
    ESTAB   0        0                10.0.2.15:61000       123.125.114.144:80


    $ nc 123.125.114.144 80 -v
    nc: connect to 123.125.114.144 port 80 (tcp) failed: Cannot assign requested address



*Experiment 2: Same target ip, different target ports*

    $ nohup nc 123.125.114.144 443 -v &
    [3] 16215
    $ nohup: ignoring input and appending output to 'nohup.out'

    $ nohup nc 123.125.114.144 443 -v &
    [4] 16216
    $ nohup: ignoring input and appending output to 'nohup.out'

    $ ss -ant |grep 10.0.2.15:61
    ESTAB   0        0                10.0.2.15:61001       123.125.114.144:443
    ESTAB   0        0                10.0.2.15:61001       123.125.114.144:80
    ESTAB   0        0                10.0.2.15:61000       123.125.114.144:443
    ESTAB   0        0                10.0.2.15:61000       123.125.114.144:80

    $ nc 123.125.114.144 443 -v
    nc: connect to 123.125.114.144 port 443 (tcp) failed: Cannot assign requested address


*Experiment 3: multiple target ip same target port*

    $ nohup nc 220.181.57.216 80 -v &
    [5] 16222
    $ nohup: ignoring input and appending output to 'nohup.out'

    $ nohup nc 220.181.57.216 80 -v &
    [6] 16223
    $ nohup: ignoring input and appending output to 'nohup.out'

    $ nc 220.181.57.216 80 -v
    nc: connect to 220.181.57.216 port 80 (tcp) failed: Cannot assign requested address

    $ ss -ant |grep :80
    SYN-SENT  0        1               10.0.2.15:61001       220.181.57.216:80
    SYN-SENT  0        1               10.0.2.15:61000       220.181.57.216:80
    SYN-SENT  0        1               10.0.2.15:61001      123.125.114.144:80
    SYN-SENT  0        1               10.0.2.15:61000      123.125.114.144:80


*Experiment 4: multiple target ip different target ports*

    $ nohup nc 123.125.114.144 80 -v &

    $ nohup nc 123.125.114.144 80 -v &

    $ nc 123.125.114.144 80 -v
    nc: connect to 123.125.114.144 port 80 (tcp) failed: Cannot assign requested address

    $ nohup nc 123.125.114.144 443 -v &

    $ nohup nc 123.125.114.144 443 -v &

    $ nc 123.125.114.144 443 -v
    nc: connect to 123.125.114.144 port 443 (tcp) failed: Cannot assign requested address

    $ nohup nc 220.181.57.216 80 -v &

    $ nohup nc 220.181.57.216 80 -v &

    $ nc 220.181.57.216 80 -v
    nc: connect to 220.181.57.216 port 80 (tcp) failed: Cannot assign requested address

    $ nohup nc 220.181.57.216 443 -v &

    $ nohup nc 220.181.57.216 443 -v &

    $ nc 220.181.57.216 443 -v
    nc: connect to 220.181.57.216 port 443 (tcp) failed: Cannot assign requested address

    $ ss -ant |grep 10.0.2.15:61
    SYN-SENT  0        1               10.0.2.15:61001       220.181.57.216:80
    ESTAB     0        0               10.0.2.15:61001      123.125.114.144:443
    ESTAB     0        0               10.0.2.15:61000       220.181.57.216:443
    SYN-SENT  0        1               10.0.2.15:61000       220.181.57.216:80
    SYN-SENT  0        1               10.0.2.15:61001      123.125.114.144:80
    ESTAB     0        0               10.0.2.15:61000      123.125.114.144:443
    SYN-SENT  0        1               10.0.2.15:61000      123.125.114.144:80
    ESTAB     0        0               10.0.2.15:61001       220.181.57.216:443

*Summarize*

So can it be said that the first statement in the preface is wrong? After checking the information, it cannot be said that the first statement is wrong:
1. The first statement is correct when the system's kernel version is less than 3.2
2. When the kernel version of the system is greater than or equal to 3.2, the second statement is correct

The previous experiments were all operated in an environment with a kernel version greater than 3.2, so the second statement was confirmed.

# Pod affinity and anti-affinity, Pod topology distribution constraints
Scattering and scheduling Pods to different places can avoid service unavailability due to factors such as software and hardware failures, fiber failures, power outages, or natural disasters, so as to achieve high-availability deployment of services.

Kubernetes supports two ways to break up pod scheduling:
* Pod Anti-Affinity
* *Pod Topology Spread Constraints

topologySpreadConstraints is more powerful than podAntiAffinity and provides finer scheduling control. We can understand that topologySpreadConstraints is an upgraded version of podAntiAffinity.

Force pods to be scattered and scheduled to different nodes (strong anti-affinity) to avoid single point of failure

    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: nginx
    spec:
    replicas: 1
    selector:
        matchLabels:
        app: nginx
    template:
        metadata:
        labels:
            app: nginx
        spec:
        affinity:
            podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
            - topologyKey: kubernetes.io/hostname
                labelSelector:
                matchLabels:
                    app: nginx
        containers:
        - name: nginx
            image: nginx

* labelSelector.matchLabelsReplace it with the label actually used by the selected Pod.
* topologyKey: The key of a certain label of the node, which can represent the topological domain of the node. Well-Known Labels can be used , and the * commonly used ones are kubernetes.io/hostname(node ​​dimension), topology.kubernetes.io/zone(availability zone/machine room dimension). You can also * manually add a custom label to the node to define the topology domain, such as rack(rack dimension), machine(physical machine dimension), switch(switch * dimension).
* If you don't want to use coercion, you can use weak anti-affinity to let Pods be scheduled to different nodes as much as possible:

    podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - podAffinityTerm:
        topologyKey: kubernetes.io/hostname
        weight: 100

Forcibly disperse and schedule Pods to different availability zones (computer rooms) to achieve cross-computer room disaster recovery

kubernetes.io/hostnameReplace topology.kubernetes.io/zonewith , and the rest are the same as above.

Disperse and schedule Pods to each node as evenly as possible

    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: nginx
    spec:
    replicas: 1
    selector:
        matchLabels:
        app: nginx
    template:
        metadata:
        labels:
            app: nginx
        spec:
        topologySpreadConstraints:
        - maxSkew: 1
            topologyKey: kubernetes.io/hostname
            whenUnsatisfiable: DoNotSchedule
            labelSelector:
            - matchLabels:
                app: nginx
        containers:
        - name: nginx
            image: nginx

* topologyKey: Similar to the configuration in podAntiAffinity.
* labelSelector: Similar to the configuration in podAntiAffinity, except that multiple groups of pod labels can be selected here.
* maxSkew: Must be an integer greater than zero, indicating the maximum value that can tolerate the difference in the number of Pods in different topology domains. Here 1 means that only 1 Pod difference is allowed.
* whenUnsatisfiable: Indicates what to do when the condition is not met. DoNotScheduleNo scheduling (keep pending), similar to strong anti-affinity; ScheduleAnywaymeans to schedule, similar to weak anti-affinity;

The above configurations are combined to explain: All nginx Pods are strictly evenly dispersed and scheduled to different nodes. The number of copies of nginx on different nodes can only differ by 1 at most. If there are nodes that cannot schedule more Pods due to other factors (such as resource Insufficient), then let the remaining nginx copy Pending.

Disperse and schedule Pods on each node as evenly as possible, without forcing (change DoNotSchedule to ScheduleAnyway):

    spec:
    topologySpreadConstraints:
    - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
        - matchLabels:
            app: nginx

If the cluster nodes support cross-availability zones, Pods can also be distributed and scheduled to each availability zone as evenly as possible to achieve a higher level of high availability (topologyKey is changed topology.kubernetes.io/zone):

    spec:
    topologySpreadConstraints:
    - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
        - matchLabels:
            app: nginx


Furthermore, while pods can be distributed and scheduled to each availability zone as evenly as possible, each node in the availability zone should also be dispersed as much as possible :

    spec:
    topologySpreadConstraints:
    - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
        - matchLabels:
            app: nginx
    - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
        - matchLabels:
            app: nginx

References:
* https://github.com/imroc/kubernetes-guide
* https://mozillazg.com/2019/05/linux-what-net.ipv4.ip_local_port_range-effect-or-mean.html
