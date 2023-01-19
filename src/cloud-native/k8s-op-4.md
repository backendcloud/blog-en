release time :2023-01-10 13:01


> The content of this article is based on https://github.com/imroc/kubernetes-guide

# Occasionally DNS resolution fails

There are many implementations of the Kubernetes cluster network, and a large part of them uses the Linux bridge: the network card of each Pod is a veth device, and the other end of the veth pair is connected to the bridge on the host machine. Since the bridge is a virtual layer-2 device, the communication between Pods on the same node is directly forwarded through the layer-2 layer, and the cross-node communication will pass through the host eth0.

Regardless of the iptables or ipvs forwarding mode, accessing the Service in Kubernetes will perform DNAT, DNAT the packet that originally accessed the ClusterIP:Port into an Endpoint (PodIP:Port) of the Service, and then the kernel will insert the connection information into the conntrack table to record the connection. When the destination end returns the packet, the kernel matches the connection from the conntrack table and reverses NAT, so that the original path returns to form a complete connection link.

But the Linux bridge is a virtual layer-2 forwarding device, and iptables conntrack is on the third layer, so if you directly access the address in the same bridge, it will go directly to the layer-2 forwarding without going through conntrack:

1. When a Pod accesses the Service, the destination IP is the Cluster IP, not the address in the bridge, and it will be forwarded through Layer 3, which will be converted into PodIP:Port by DNAT.
2. If the DNAT is forwarded to the pod on the same node, and the destination pod finds that the destination IP is on the same bridge when the destination pod returns the packet, it will directly forward it through the second layer without calling conntrack, resulting in no return of the original path when returning the packet.


Since there is no return path, the communication between the client and the server is not on the same "channel", and they do not think they are in the same connection, so they cannot communicate normally.

A common problem is that DNS resolution fails occasionally. When the pod on the node where coredns is located resolves dns, the dns request falls on the coredns pod of the current node. This problem may occur.

The kernel parameter bridge-nf-call-iptables (set to 1) indicates that the bridge device also calls the layer-3 rules configured by iptables (including conntrack) when forwarding at layer-2, so enabling this parameter can solve the communication between the above-mentioned Service and the node problem, which is why in the Kubernetes environment, bridge-nf-call-iptables is mostly required to be enabled.

    sysctl -w net.bridge.bridge-nf-call-iptables=1

# offline deployment
The domestic network environment makes it impossible to download foreign images, or the deployment environment cannot be connected to the external network. At this time, offline deployment is required. For the part of the container image, you can synchronize the required images in the public image repository to the private image repository.

skepeo is an open source container image handling tool, which is relatively general and supported by various image warehouses.

Organize mirror list. for example:

    $ helm template -n monitoring -f kube-prometheus-stack.yaml ./kube-prometheus-stack | grep "image:" | awk -F 'image:' '{print $2}' | awk '{$1=$1;print}' | sed -e 's/^"//' -e 's/"$//' > images.txt
    $ cat images.txt
    quay.io/prometheus/node-exporter:v1.3.1
    quay.io/kiwigrid/k8s-sidecar:1.19.2
    quay.io/kiwigrid/k8s-sidecar:1.19.2
    grafana/grafana:9.0.2
    registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.5.0
    quay.io/prometheus-operator/prometheus-operator:v0.57.0
    quay.io/prometheus/alertmanager:v0.24.0
    quay.io/prometheus/prometheus:v2.36.1
    bats/bats:v1.4.1
    k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1
    k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1

Prepare sync mirror script:

    #! /bin/bash

    DST_IMAGE_REPO="127.0.0.1:5000/prometheus"

    cat images.txt | while read line
    do
            # 若同步失败一直尝试直到成功，这里可以优化下，配置下最大尝试次数
            while :
            do
                    skopeo sync --src=docker --dest=docker $line $DST_IMAGE_REPO
                    if [ "$?" == "0" ]; then
                            break
                    fi
            done
    done


Before executing the above synchronous mirroring script, if the mirror warehouse requires authentication, you must first log in. Both source and destination.

The login method is very simple, just like docker login, specify the mirror warehouse address to log in:

    skopeo login registry

Then enter the username and password.

# graceful abort
The process of pod termination:

1. The Pod is deleted and the state changes to Terminating. The deletionTimestamp field in the Pod metadata in the pod's yaml information will be marked with the deletion time.

2. The kube-proxy watch starts to update the forwarding rules of iptables or ipvs when the deletion time is marked, removes the Pod from the endpoint list of the service, and new traffic is no longer forwarded to the Pod.

3. When the kubelet watch reaches the deletion time and is marked, the kubelet destroys the Pod process.

3.1. If there is a container configured with preStop Hook in the Pod, it will be executed.

3.2. Send the SIGTERM signal to the main process in the container to notify the container process to start a graceful stop.

3.3. Wait for the main process in the container to stop completely. If it has not stopped completely within terminationGracePeriodSeconds (default 30s), send a SIGKILL signal to kill it forcibly.

3.4. All container processes are terminated and Pod resources are cleaned up.

3.5. Notify the APIServer that the Pod is destroyed and the Pod is deleted.

Kubernetes is only responsible for sending the SIGTERM signal to the main process in the container. If the business process is not the main process, the signal cannot be obtained, so it cannot be terminated gracefully. If the business process is the main process, but the business code does not receive the processing logic of the SIGTERM signal, it cannot be terminated gracefully. At this time, you can use the preStop Hook to wait for a period of time or do some cleaning work before the suspension, and replace the graceful suspension at the business code level with the graceful suspension at the configuration level. for example:

```yaml
lifecycle:
  preStop:
    exec:
      command:
      - /clean.sh
```

```yaml
lifecycle:
  preStop:
    exec:
      command:
      - sleep
      - 5s
```

The business code processes the SIGTERM signal, which is to obtain the signal and execute the business logic of drainage. kube-proxy no longer forwards new traffic to the Pod, as long as the existing traffic is processed, use the metaphor of drainage.

Below are code samples for several languages ​​to get the SIGTERM signal.

*shell*

    #!/bin/sh

    ## Redirecting Filehanders
    ln -sf /proc/$$/fd/1 /log/stdout.log
    ln -sf /proc/$$/fd/2 /log/stderr.log

    ## Pre execution handler
    pre_execution_handler() {
    ## Pre Execution
    # TODO: put your pre execution steps here
    : # delete this nop
    }

    ## Post execution handler
    post_execution_handler() {
    ## Post Execution
    # TODO: put your post execution steps here
    : # delete this nop
    }

    ## Sigterm Handler
    sigterm_handler() { 
    if [ $pid -ne 0 ]; then
        # the above if statement is important because it ensures 
        # that the application has already started. without it you
        # could attempt cleanup steps if the application failed to
        # start, causing errors.
        kill -15 "$pid"
        wait "$pid"
        post_execution_handler
    fi
    exit 143; # 128 + 15 -- SIGTERM
    }

    ## Setup signal trap
    # on callback execute the specified handler
    trap 'sigterm_handler' SIGTERM

    ## Initialization
    pre_execution_handler

    ## Start Process
    # run process in background and record PID
    >/log/stdout.log 2>/log/stderr.log "$@" &
    pid="$!"
    # Application can log to stdout/stderr, /log/stdout.log or /log/stderr.log

    ## Wait forever until app dies
    wait "$pid"
    return_code="$?"

    ## Cleanup
    post_execution_handler
    # echo the return code of the application
    exit $return_code


*go*

    package main

    import (
        "fmt"
        "os"
        "os/signal"
        "syscall"
    )

    func main() {

        sigs := make(chan os.Signal, 1)
        done := make(chan bool, 1)
        //registers the channel
        signal.Notify(sigs, syscall.SIGTERM)

        go func() {
            sig := <-sigs
            fmt.Println("Caught SIGTERM, shutting down")
            // Finish any outstanding requests, then...
            done <- true
        }()

        fmt.Println("Starting application")
        // Main logic goes here
        <-done
        fmt.Println("exiting")
    }

*python*

```python
import signal, time, os

def shutdown(signum, frame):
    print('Caught SIGTERM, shutting down')
    # Finish any outstanding requests, then...
    exit(0)

if __name__ == '__main__':
    # Register handler
    signal.signal(signal.SIGTERM, shutdown)
    # Main logic goes here
```

*nodejs*

    process.on('SIGTERM', () => {
    console.log('The service is about to shut down!');
    
    // Finish any outstanding requests, then...
    process.exit(0); 
    });


*java*

```java
import sun.misc.Signal;
import sun.misc.SignalHandler;
 
public class ExampleSignalHandler {
    public static void main(String... args) throws InterruptedException {
        final long start = System.nanoTime();
        Signal.handle(new Signal("TERM"), new SignalHandler() {
            public void handle(Signal sig) {
                System.out.format("\nProgram execution took %f seconds\n", (System.nanoTime() - start) / 1e9f);
                System.exit(0);
            }
        });
        int counter = 0;
        while(true) {
            System.out.println(counter++);
            Thread.sleep(500);
        }
    }
}
```

One of the reasons why there is no graceful termination mentioned above is that the business logic is not the main process, often because the startup entry like /bin/sh -c my-app is used. Or use a script file like /entrypoint.sh as the entry point, and start the business process in the script. The main process of the container is the shell, and the business process is started in the shell and becomes a child process of the shell process. Without special configuration, the shell will not pass the SIGTERM signal to its own child process, so that the business process will not trigger the stop logic. At this time, we can only wait until K8S gracefully stops the timeout (terminationGracePeriodSeconds, default 30s), and send SIGKILL to forcibly kill the shell and its child processes.

**How to solve the problem that the business process cannot obtain the signal**
1. Try not to use the shell to start the business process, start the business process directly
2. If it must be started through the shell, a certain configuration is required to transmit the signal in the SHELL.


Signals are transmitted in SHELL. Specifically, there are three ways.

1. Start with exec

Add an exec before the command to start the binary in the shell to make the process started by the binary replace the current shell process, that is, to make the newly started process the main process:

    #! /bin/bash
    ...

    exec /bin/yourapp 


2. Multi-process scenario: use trap to pass signals

Multiple business processes need to be started in a single container. At this time, they can only be started through the shell, but the above exec method cannot be used to transmit signals, because exec can only make one process replace the current shell as the main process.

At this time, we can use trap in the shell to capture the signal. When the signal is received, the callback function is triggered to pass the signal to the business process through kill. The script example:

    #! /bin/bash

    /bin/app1 & pid1="$!" 
    echo "app1 started with pid $pid1"

    /bin/app2 & pid2="$!" 
    echo "app2 started with pid $pid2"

    handle_sigterm() {
    echo "[INFO] Received SIGTERM"
    kill -SIGTERM $pid1 $pid2 
    wait $pid1 $pid2 
    }
    trap handle_sigterm SIGTERM 

    wait

3. The perfect solution: use the init system

The previous solution is actually to implement a minimal init system (or supervisor) with scripts to manage all sub-processes, but its logic is very simple. It simply transparently transmits specified signals to sub-processes. In fact, the community has more perfect ones. solution, both dumb-init and tini can be used as the init process, started as the main process (PID 1) in the container, and then it runs the shell to execute the script we specified (the shell is used as a sub-process), and the business process started in the shell It also becomes its child process, and when it receives a signal, it will pass it to all child processes, which can perfectly solve the problem that SHELL cannot deliver signals, and has the ability to recycle zombie processes.

This is an example of a Dockerfile that uses dumb-init as an example to create an image:


    FROM ubuntu:22.04
    RUN apt-get update && apt-get install -y dumb-init
    ADD start.sh /
    ADD app1 /bin/app1
    ADD app2 /bin/app2
    ENTRYPOINT ["dumb-init", "--"]
    CMD ["/start.sh"]

This is an example Dockerfile for making an image using tini as an example:

    FROM ubuntu:22.04
    ENV TINI_VERSION v0.19.0
    ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini /tini
    COPY entrypoint.sh /entrypoint.sh
    RUN chmod +x /tini /entrypoint.sh
    ENTRYPOINT ["/tini", "--"]
    CMD [ "/start.sh" ]


Example start.sh script:

    #! /bin/bash
    /bin/app1 &
    /bin/app2 &
    wait

> The above operations are all remedial operations that do not follow the principle that a single pod of a microservice only runs a single business process.

All of the above solutions are optimized for the drainage of the valves that have been turned off. But in fact, there is another situation that needs to be optimized at the service level of k8s, that is, the stock request to the stock connection on the Pod has not been processed, and if the endpoint is removed (unbound) directly, the request may be abnormal. At this time, what is expected is to wait for the stock request to be processed before actually unbinding the old Pod.

Now mainstream cloud vendors not only support kube-proxy forwarding to pods, but also support LB direct access to pods, that is, LBs directly forward traffic to pods, without the need for another forwarding in the cluster.

Tencent Cloud TKE officially provides solutions for both Layer 4 Service and Layer 7 Ingress.

If it is a four-tier Service, just add such annotations to the Service (provided that the Service uses the CLB direct-to-Pod mode):

service.cloud.tencent.com/enable-grace-shutdown: "true"

> Refer to the official document Service Graceful Shutdown


If it is a seven-layer CLB type Ingress, just add this annotation to the Ingress (provided that the Service uses the CLB direct Pod mode):

ingress.cloud.tencent.com/enable-grace-shutdown: "true"

> Refer to the official document Ingress graceful downtime

Alibaba Cloud ACK currently only provides solutions for four-tier services, enabling graceful interruption and setting the interruption timeout through annotations:

service.beta.kubernetes.io/alibaba-cloud-loadbalancer-connection-drain: “on”

service.beta.kubernetes.io/alibaba-cloud-loadbalancer-connection-drain-timeout: “900”

> Refer to the official document to configure load balancing through Annotation

# Smooth workload upgrade
Two embarrassing situations occur:
1. The old copy will be destroyed soon, and the kube-proxy node where the client is located has not yet updated the forwarding rules, and still schedules the new connection to the old copy, causing connection exceptions, and may report "connection refused" (during the process stop process, no longer accepting new request) or "no route to host" (the container has been completely destroyed, the network card and IP no longer exist).
2. The new copy starts, and the node kube-proxy where the client is located quickly watches the new copy, updates the forwarding rules, and dispatches the new connection to the new copy, but the process in the container starts very slowly (such as the java process like Tomcat), and still During the startup process, the port has not been monitored, the connection cannot be processed, and the connection is abnormal. Usually, the "connection refused" error will be reported.

For the first case, you can add preStop to the container, let the Pod sleep for a period of time before it is actually destroyed, wait for the kube-proxy node where the client is located to update the forwarding rules, and then actually destroy the container. This ensures that the Pod can continue to run normally for a period of time after the Pod is terminated. During this period, if new requests are forwarded because the forwarding rules on the client side are not updated in time, the Pod can still process the requests normally, avoiding connection exceptions. It sounds a bit inelegant, but the actual effect is quite good. There is no silver bullet in the distributed world. We can only try our best to find and practice the optimal solution that can solve the problem under the current design status.

For the second case, you can add a ReadinessProbe (readiness check) to the container, so that the process in the container is actually started to update the Endpoint of the Service, and then the node kube-proxy where the client is located updates the forwarding rules to allow traffic to enter. This can ensure that the traffic will not be forwarded until the Pod is fully ready, which also avoids the occurrence of link exceptions.

yaml example:

```yaml
readinessProbe:
  httpGet:
    path: /healthz
    port: 80
    httpHeaders:
    - name: X-Custom-Header
      value: Awesome
  initialDelaySeconds: 10
  timeoutSeconds: 1
lifecycle:
  preStop:
    exec:
      command: ["/bin/bash", "-c", "sleep 10"]
```

Finally, the business itself also needs to achieve graceful termination to avoid interrupting the business when it is destroyed. Refer to the graceful termination in this article.