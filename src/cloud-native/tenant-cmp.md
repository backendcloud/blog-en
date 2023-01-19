release time :2021-09-13 14:22





# Implementation plan
To create a resource pool tenant in the Java version of the cloud management platform project, the southbound interface needs to create a namespace for the underlying Kubernetes and create a Kubernetes user with the same name.

Requests to the k8s apiserver (kubectl client, client library, or constructing REST requests to access the kubernetes API).

The Java version of the Kubernetes client library has the official version io.kubernetes.client and the unofficial io.fabric8.kubernetes. The latter's unofficial ones are stronger than the official ones, so the plan is to choose unofficial ones. However, after investigation, it is found that even unofficial interfaces have not been implemented. For example, the "kubectl config set-context" that needs to be used in the requirements solution does not correspond to the library, but the interface is implemented. Refer to the discussion in the github community https://github.com/fabric8io/kubernetes-client/issues/1512 These interface client libraries have not been implemented. 



```java
    public static void nsCreateOrReplace(String yamlString, String name, String quota) throws IOException {
        KubernetesClient k8sClient = KubernetesClientGenerator.fromYamlStringGetKubernetesClient(yamlString);
        Namespace ns = new NamespaceBuilder().withNewMetadata().withName(name).endMetadata().build();
        k8sClient.namespaces().createOrReplace(ns);
        ResourceQuota resourceQuota = new ResourceQuotaBuilder()
                .withNewMetadata().withName("ns-quota").endMetadata()
                .withNewSpec()
                .addToHard("cpu", new Quantity("0"))
                .addToHard("memory", new Quantity("0"))
                .addToHard("pods", new Quantity("0"))
                .endSpec().build();
        k8sClient.resourceQuotas().inNamespace(name).createOrReplace(resourceQuota);
        if (quota == null) {return;}
        Gson gson = new Gson();
        ListVDCDetailResponse.PoolTenantsDTO.QuotaSpecDTO quotaSpec = gson.fromJson(quota, ListVDCDetailResponse.PoolTenantsDTO.QuotaSpecDTO.class);
//        if (cpu == null) {cpu = 0;}
//        if (memory == null) {memory = 0;}
//        if (pods ==  null) {pods = 0;}
        if (quotaSpec == null) {return;}
        resourceQuota = new ResourceQuotaBuilder()
                .withNewMetadata().withName("ns-quota").endMetadata()
                .withNewSpec()
                .addToHard("cpu", new Quantity(quotaSpec.getCpu().toString()))
                .addToHard("memory", new Quantity(quotaSpec.getMemory().toString()))
                .addToHard("pods", new Quantity(quotaSpec.getPods().toString()))
                .endSpec().build();
        k8sClient.resourceQuotas().inNamespace(name).createOrReplace(resourceQuota);
    }
```

```java
public static void nsDelete(String yamlString, String name) throws IOException {
    KubernetesClient k8sClient = KubernetesClientGenerator.fromYamlStringGetKubernetesClient(yamlString);
    Namespace ns = new NamespaceBuilder().withNewMetadata().withName(name).endMetadata().build();
    k8sClient.namespaces().delete(ns);
}
```

```java
public static String createKubeConfig(String cluster, String namespace) throws IOException {
    File file = new File("/etc/cmp/kubeconfig/create-kubeconfig.sh");
    if(file.exists()) {
        String tt = "/etc/cmp/kubeconfig/create-kubeconfig.sh " + cluster + " " + namespace + " 2>&1";
        String[] cmd = new String[] { "/bin/bash", "-c", tt};
        Process ps = Runtime.getRuntime().exec(cmd);
        BufferedReader br = new BufferedReader(new InputStreamReader(ps.getInputStream()));
        StringBuffer sb = new StringBuffer();
        String line;
        while ((line = br.readLine()) != null) {
            sb.append(line).append("\n");
        }
        String result = sb.toString();
        log.debug(tt);
        log.debug(result);
        String configFilePath = "/etc/cmp/kubeconfig/" + cluster + "/" + namespace + ".config";
        File configFile = new File(configFilePath);
        if(configFile.exists()) {
            BufferedReader in = new BufferedReader(new FileReader(configFilePath));
            String str;
            StringBuffer sb2 = new StringBuffer();
            while ((str = in.readLine()) != null) {
                sb2.append(str).append("\n");
            }
            String result2 = sb2.toString();
            log.debug("kube config:");
            log.debug(result2);
            return result2;
        }
    }
    return null;
}
```

```java
public static void deleteKubeConfig(String cluster, String namespace) throws IOException {
    File file = new File("/etc/cmp/kubeconfig/delete-kubeconfig.sh");
    if(file.exists()) {
        String[] cmd = new String[] { "/bin/bash", "-c", "/etc/cmp/kubeconfig/delete-kubeconfig.sh " + cluster + " " + namespace};
        Process ps = Runtime.getRuntime().exec(cmd);
        BufferedReader br = new BufferedReader(new InputStreamReader(ps.getInputStream()));
        StringBuffer sb = new StringBuffer();
        String line;
        while ((line = br.readLine()) != null) {
            sb.append(line).append("\n");
        }
        String result = sb.toString();
        log.info(result);
    }
}
```

The functions realized by the above four functions are: create ns, delete ns, create kubeconfig, delete kubeconfig.

Two shell scripts are called in the java program that creates and deletes kubeconfig create-kubeconfig.shand delete-kubeconfig.shis used to create and delete kubeconfig respectively.

    # cat create-kubeconfig.sh
    set -v
    [[ $# -ne 2 ]] && usage

    mkdir -p /etc/cmp/kubeconfig/$1/
    cd /etc/cmp/kubeconfig/$1/
    openssl genrsa -out $2.key 2048
    openssl req -new -key $2.key -out $2.csr -subj "/CN=$2/O=$2"
    openssl x509 -req -in $2.csr -CA kubernetes/ca.crt -CAkey kubernetes/ca.key -CAcreateserial -out $2.crt -days 36500
    kubectl config set-credentials $2 --client-certificate=$2.crt  --client-key=$2.key
    kubectl config set-context $2 --cluster=$1 --namespace=$2 --user=$2
    kubectl config view --flatten --merge --minify --context $2 > $2.config
    #kubectl create ns $2

    cat>$2-rolebinding.yaml<<EOF
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
    name: $2-rolebinding
    namespace: $2
    subjects:
    - kind: User
    name: $2
    apiGroup: ""
    roleRef:
    kind: ClusterRole
    name: admin
    apiGroup: ""
    EOF

    kubectl apply -f $2-rolebinding.yaml

    usage(){
        echo "Usage: $0 <cluster-name> <namespace-name>"
        exit 1
    }


```bash
# cat delete-kubeconfig.sh
set -v
if [ $# -ne 2 ];
then
    exit
fi

cd /etc/cmp/kubeconfig/$1/
kubectl delete -f $2-rolebinding.yaml
kubectl config delete-context $2
kubectl config delete-user $2
rm -rf $2-rolebinding.yaml $2.config $2.key $2.crt $2.csr
#kubectl delete ns $2
```


## Create user credentials
Kubernetes does not have a User Account API object. To create a user account, use a private key assigned by the administrator to create it. Refer to the method in the official document to create a User with an OpenSSL certificate, or use the simpler cfssl tool to create:

Create a private key for user xxx and name it: xxx.key:

    $ openssl genrsa -out xxx.key 2048

Use the private key you just created to create a certificate signing request file: xxx.csr, be sure to specify the user name and group in the -subj parameter (CN means user name, O means group):

    $ openssl req -new -key xxx.key -out xxx.csr -subj "/CN=xxx/O=xxx"

Then find the CA of the Kubernetes cluster. If the cluster installed by kubeadm is used, the CA-related certificate is located under the /etc/kubernetes/ssl/ directory. If it is built in binary mode, the CA directory has been specified when the cluster was first built. Use the two files ca.crt and ca.key under this directory to approve the above certificate request

Generate the final certificate file, here we set the validity period of the certificate to 500 days:

    $ openssl x509 -req -in xxx.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out xxx.crt -days 500

    Now check whether a certificate file is generated under the current folder:

    $ ls
    xxx.csr xxx.key xxx.crt

Now you can create a new credential and context in the cluster using the certificate file and private key file you just created:

    $ kubectl config set-credentials xxx –client-certificate=xxx.crt –client-key=xxx.key

You can see that a user xxx has been created, and then set a new Context for this user:

    $ kubectl config set-context xxx –cluster=kubernetes –namespace=kube-system –user=xxx

At this point, the user haimaxy has been successfully created. Now when using the current configuration file to operate the kubectl command, an error should occur, because no operation permissions have been defined for the user:

    $ kubectl get pods –context=xxx

Error from server (Forbidden): pods is forbidden: User “xxx” cannot list pods in the namespace “default”

## Create roles and role permission bindings
Role and ClusterRole: role and cluster role, both objects contain the above Rules element, the difference between the two is that in Role, the defined rules are only applicable to a single namespace, which is associated with namespace, while ClusterRole is Cluster-wide, so defined rules are not bound by namespaces.

After the user is created, it is necessary to add operation permissions to the user. Let's define a YAML file and create a role that allows the user to operate Deployment, Pod, and ReplicaSets, as defined below: (xxx-role.yaml)

    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
    name: xxx-role
    namespace: kube-system
    rules:
    - apiGroups: ["", "extensions", "apps"]
    resources: ["deployments", "replicasets", "pods"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"] # 也可以使用['*']

Among them, Pod belongs to the core API Group, and you can use empty characters in YAML, while Deployment belongs to the apps API Group, and ReplicaSets belongs to the extensions API Group, so apiGroups under rules integrates the API Groups of these resources: [" ", "extensions", "apps"], where verbs can perform operations on these resource objects, all operation methods are required, and ['*'] can also be used instead.

Then create this Role:

$ kubectl create -f xxx-role.yaml

Note that the context of xxx above is not used here, because there is no permission

The Role is created. This Role has nothing to do with the user xxx. You need to create a RoleBinding object, and bind the above xxx-role role to the user xxx under the kube-system namespace: (xxx-rolebinding.yaml)

    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
    name: xxx-rolebinding
    namespace: kube-system
    subjects:
    - kind: User
    name: xxx
    apiGroup: ""
    roleRef:
    kind: Role
    name: xxx-role
    apiGroup: “"

The subjects keyword in the above YAML file is the object mentioned above to try to operate the cluster, which corresponds to the above User account xxx, and use kubectl to create the above resource object:

    $ kubectl create -f xxx-rolebinding.yaml

test

Now you should be able to operate the cluster with the xxx context above:

    $ kubectl get pods --context=xxx
    ....

It can be seen that the use of kubectl does not specify a namespace. This is because we have assigned permissions to the user, and we can view all pods in the assigned namespace. If we add a -n default at the end

    $ kubectl –context=xxx-context get pods –namespace=default

Error from server (Forbidden): pods is forbidden: User “xxx” cannot list pods in the namespace “default”
is as expected. Because the user does not have the operation permission of the default namespace






















