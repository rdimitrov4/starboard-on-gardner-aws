
Starboard integrates security tools into the Kubernetes environment, so that users can find and view the risks that relate to different resources in a Kubernetes-native way.

https://github.com/aquasecurity/starboard

### Why use Starboard:

- Vulnerability information in your running workloads, scanned using Trivy
- Workload audits provided by Fairwinds Polaris
- CIS benchmark results per node provided by kube-bench
- Pen-testing results provided by kube-hunter

### Starboard can be run in two different modes:

- As a command, so you can trigger scans and view the risks in a kubectl-compatible way or as part of your CI/CD pipeline.
- As an operator to automatically update security reports in response to workload and other changes on a Kubernetes
  cluster - for example, initiating a vulnerability scan when a new pod is started.

#

1) **Connect the local kubectl on current shell instance to the Gardner cluster with a pre-generated token.**

    `export KUBECONFIG=$KUBECONFIG:/path/to/file/kubeconfig--lab.yaml`

2) **Setup Istio on the cluster**
- Platform setup for Gardner: https://istio.io/latest/docs/setup/platform-setup/gardener/ (networking Calico for further networking policy testing)

- Install Istio using istioctl https://istio.io/latest/docs/setup/getting-started/

- Download Istio

    Go to the Istio release page to download the installation file for your OS, or download and extract the latest release automatically (Linux or macOS):
    
    `$ curl -L https://istio.io/downloadIstio | sh -`

    The command above downloads the latest release (numerically) of Istio. You can pass variables on the command line to download a specific version or to override the processor architecture. For example, to download Istio 1.6.8 for the x86_64 architecture, run:
    
    `$ curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.6.8 TARGET_ARCH=x86_64 sh -`

    Move to the Istio package directory. For example, if the package is istio-1.11.3:
    
    `$ cd istio-1.11.3`

    The installation directory contains:
    
    `Sample applications in samples/`
    `The istioctl client binary in the bin/ directory.`

    Add the istioctl client to your path (Linux or macOS):
    
    `$ export PATH=$PWD/bin:$PATH`

- Install Istio

    `$ istioctl install`

    Add a namespace label to instruct Istio to automatically inject Envoy sidecar proxies when you deploy your application later:
    `$ kubectl label namespace default istio-injection=enabled`
    `namespace/default labeled`


3) **Check the doplyed resources on default, istio-system and kube-system namespaces**
    
    `kubectl get all; kubectl get all -n istio-system; kubectl get all -n kube-system`
    
4) **Get Starboard**

    https://github.com/aquasecurity/starboard
    
5) **Install Starboard CLI from binary releases**
- Download your desired version https://github.com/aquasecurity/starboard/releases
- Unpack it (tar -zxvf starboard_linux_x86_64.tar.gz)
- Find the starboard binary in the unpacked directory, and move it to its desired destination (In my case $sudo mv starboard_darwin_x86_64/starboard /bin)

6) **Install Starboard Operator**

     1) Send custom resource definitions to the Kubernetes API:

     ```
     kubectl apply -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/crd/vulnerabilityreports.crd.yaml \
    -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/crd/configauditreports.crd.yaml \
    -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/crd/clusterconfigauditreports.crd.yaml \
    -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/crd/ciskubebenchreports.crd.yaml
    ```
  
     2) Send the following Kubernetes objects definitions to the Kubernetes API:

     ```
     kubectl apply -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/01-starboard-operator.ns.yaml \
    -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/02-starboard-operator.sa.yaml \
    -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/03-starboard-operator.clusterrole.yaml \
    -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/04-starboard-operator.clusterrolebinding.yaml
    ```

     3) (Optional) Configure Starboard by creating the starboard ConfigMap and the starboard secret in the starboard-operator namespace.
    
    `kubectl apply -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/05-starboard-operator.config.yaml`
    
     4) Finally, create the starboard-operator Deployment in the starboard-operator namespace to start the operator's pod:
     
    `kubectl apply -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/06-starboard-operator.deployment.yaml`
    
    5) To confirm that the operator is running, check the number of replicas created by the starboard-operator Deployment in the starboard-operator namespace:
    
    `$ kubectl get deployment -n starboard-operator`
    
    **You can uninstall the operator with the following command:**
    
    ```
     kubectl delete -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/06-starboard-operator.deployment.yaml \
    -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/05-starboard-operator.config.yaml \
    -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/04-starboard-operator.clusterrolebinding.yaml \
    -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/03-starboard-operator.clusterrole.yaml \
    -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/02-starboard-operator.sa.yaml \
    -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/01-starboard-operator.ns.yaml
    ```

8) **Scanning Workloads**

    The easiest way to get started with Starboard is to use an imperative starboard command, which allows ad hoc scanning of Kubernetes workloads deployed in your cluster.
    To begin with, execute the following one-time setup command:

    `starboard init`

    The `init` subcommand creates the `starboard` namespace, in which Starboard executes Kubernetes jobs to perform scans. It also sends custom security resources definitions to the Kubernetes API:
    
    ```
    $ kubectl api-resources --api-group aquasecurity.github.io
    
    NAME                        SHORTNAMES           APIVERSION                        NAMESPACED   KIND
    ciskubebenchreports         kubebench            aquasecurity.github.io/v1alpha1   false        CISKubeBenchReport
    clusterconfigauditreports   clusterconfigaudit   aquasecurity.github.io/v1alpha1   false        ClusterConfigAuditReport
    configauditreports          configaudit          aquasecurity.github.io/v1alpha1   true         ConfigAuditReport
    kubehunterreports           kubehunter           aquasecurity.github.io/v1alpha1   false        KubeHunterReport
    vulnerabilityreports        vuln,vulns           aquasecurity.github.io/v1alpha1   true         VulnerabilityReport
    ```
    
    **There's also a `starboard cleanup` subcommand, which can be used to remove all resources created by Starboard.** 

    #
    
    As an example let's run in the current namespace an old version of nginx that we know has vulnerabilities:
    
    `kubectl create deployment nginx --image nginx:1.16`
    
    Run the vulnerability scanner to generate vulnerability reports:

    `starboard scan vulnerabilityreports deployment/nginx`
    
    Behind the scenes, by default this uses Trivy in Standalone mode to identify vulnerabilities in the container images associated with the specified deployment.     Once this has been done, you can retrieve the latest vulnerability reports for this workload:

    `starboard get vulnerabilities deployment/nginx -o yaml`
    
    #
    **Note**
    
    Starboard relies on labels and label selectors to associate vulnerability reports with the specified Deployment. For a Deployment with N container images         Starboard creates N instances of vulnerabilityreports.aquasecurity.github.io resources. In addition, each instance has the starboard.container.name label to       associate it with a particular container's image. This means that the same data retrieved by the starboard get vulnerabilities subcommand can be fetched with     the standard kubectl get command:
    ```
    $ kubectl get vulnerabilityreports -o wide \
    >  -l starboard.resource.kind=Deployment,starboard.resource.name=nginx
    NAME                     REPOSITORY      TAG    SCANNER   AGE    CRITICAL   HIGH   MEDIUM   LOW   UNKNOWN
    deployment-nginx-nginx   library/nginx   1.16   Trivy     2m6s   3          40     24       90    0
    ```
    #
    
    Llet's take the same nginx Deployment and audit its Kubernetes configuration. As you remember we've created it with the kubectl create deployment command which applies the default settings to the deployment descriptors. However, we also know that in Kubernetes the defaults are usually the least secure.

    Run the scanner to audit the configuration using Polaris, which is the default configuration checker:

    `starboard scan configauditreports deployment/nginx`
    
    
    Retrieve the configuration audit report:
    
    `starboard get configaudit deployment/nginx -o yaml`
    
    or
    
    ```
    $ kubectl get configauditreport -o wide \
    >  -l starboard.resource.kind=Deployment,starboard.resource.name=nginx
    NAME               SCANNER   AGE   DANGER   WARNING   PASS
    deployment-nginx   Polaris   5s    0        8         9
    ```

    Once you scanned the nginx Deployment for vulnerabilities and checked its configuration you can generate an HTML report of identified risks:

    `starboard get report deployment/nginx > nginx.deploy.html`
    
    The file is saved at the `/home/user` directroy.

9) **Workloads Scanning in Operator Mode**
    
    We will again use nginx Deployment that we know is vulnerable:
    
    `kubectl create deployment nginx --image nginx:1.16`
    
    When the first ReplicaSet controlled by the `nginx` Deployment is created, the operator immediately detects that and creates the Kubernetes Job in the `starboard-operator` namespace to scan the `nginx:1.16` image for vulnerabilities. It also creates the Job to audit the Deployment's configuration for common pitfalls such as running the `nginx` container as root:
    
    ```
    $ kubectl get job -n starboard-operator
    NAME                                 COMPLETIONS   DURATION   AGE
    scan-configauditreport-c4956cb9d     0/1           1s         1s
    scan-vulnerabilityreport-c4956cb9d   0/1           1s         1s
    ```
    
    If everything goes fine, the scan Jobs are deleted, and the operator saves scan reports as custom resources in the `default namespace`, named after the Deployment's active ReplicaSet. For image vulnerability scans, the operator creates a VulnerabilityReport for each different container defined in the active ReplicaSet. In this example there is just one container image called nginx:
    
    ```
    $ kubectl get vulnerabilityreports -o wide
    NAME                                            REPOSITORY      TAG    SCANNER   AGE    CRITICAL   HIGH   MEDIUM   LOW   UNKNOWN
    replicaset-nginx-6d4cf56db6-nginx               library/nginx   1.16   Trivy     115s   25         85     84       15    0
    replicationcontroller-redis-slave-redis-slave   redis-slave     v2     Trivy     14m     1          78     758      499   0
    ```
    
    Similarly, the operator creates a ConfigAuditReport holding the result of auditing the configuration of the active ReplicaSet controlled by the nginx Deployment(also for other Replication controllers):
    
    ```
    $ kubectl get configauditreports -o wide
    NAME                                 SCANNER   AGE     DANGER   WARNING   PASS
    replicaset-nginx-6d4cf56db6          Polaris   4m18s   1        9         7
    replicationcontroller-guestbook      Polaris   13m     1        9         7
    replicationcontroller-redis-master   Polaris   13m     1        9         7
    replicationcontroller-redis-slave    Polaris   13m     1        9         7
    ```
    
    Notice that scan reports generated by the operator are controlled by Kubernetes workloads. In our example, VulnerabilityReport and ConfigAuditReport objects are controlled by the active ReplicaSet of the nginx Deployment and Guestbook Go App:
    
    ```
    $ kubectl tree deploy nginx
    NAMESPACE  NAME                                                       READY  REASON  AGE
    default    Deployment/nginx                                           -              35m
    default    └─ReplicaSet/nginx-6d4cf56db6                              -              35m
    default      ├─ConfigAuditReport/replicaset-nginx-6d4cf56db6          -              35m
    default      ├─Pod/nginx-6d4cf56db6-vzjds                             True           35m
    default      └─VulnerabilityReport/replicaset-nginx-6d4cf56db6-nginx  -              34m
    ```
    
    ```
    The tree command is a kubectl plugin to browse Kubernetes object hierarchies as a tree.
    https://github.com/ahmetb/kubectl-tree
    ```
    
    Let's update the container image of the nginx Deployment from nginx:1.16 to nginx:1.17. This will trigger a rolling update of the Deployment and eventually create another ReplicaSet.

    `kubectl set image deployment nginx nginx=nginx:1.17`
    
    Even this time the operator will pick up changes and rescan our Deployment with updated configuration:
    
    ```
    $ kubectl tree deploy nginx
    NAMESPACE  NAME                                                       READY  REASON  AGE
    default    Deployment/nginx                                           -              41m
    default    ├─ReplicaSet/nginx-6d4cf56db6                              -              41m
    default    │ ├─ConfigAuditReport/replicaset-nginx-6d4cf56db6          -              41m
    default    │ └─VulnerabilityReport/replicaset-nginx-6d4cf56db6-nginx  -              40m
    default    └─ReplicaSet/nginx-db749865c                               -              87s
    default      ├─ConfigAuditReport/replicaset-nginx-db749865c           -              82s
    default      ├─Pod/nginx-db749865c-zl248                              True           87s
    default      └─VulnerabilityReport/replicaset-nginx-db749865c-nginx   -              67s
    ```
    
    Notice that scaling up the nginx Deployment will not schedule new scan Jobs because all replica Pods refer to the same Pod templated defined by the ReplicaSet.
    
    Finally, when you delete the nginx Deployment, orphaned security reports will be deleted in the background by the Kubernetes garbage collection controller.
    
    `kubectl delete deploy nginx`
    
    ```
    kubectl get vuln,configaudit
    No resources found in default namespace.
    ```
    
10) **Infrastructure Scanning**

    The operator also discovers Kubernetes nodes and runs CIS Kubernetes Benchmark checks on each of them. The results are stored as CISKubeBenchReport objects. In other words, for each node in a cluster the operator will eventually create a benchmark report:
    
    ```
    $ kubectl get node
    NAME                                             STATUS   ROLES    AGE     VERSION
    ip-10-250-20-146.eu-central-1.compute.internal   Ready    <none>   6h23m   v1.21.5
    ip-10-250-25-164.eu-central-1.compute.internal   Ready    <none>   6h23m   v1.21.5
    ```
    
    ```
    $ kubectl get ciskubebenchreports -o wide
    NAME                                             SCANNER      AGE     FAIL   WARN   INFO   PASS
    ip-10-250-20-146.eu-central-1.compute.internal   kube-bench   6h24m   5      30     0      12
    ip-10-250-25-164.eu-central-1.compute.internal   kube-bench   6h24m   5      30     0      12
    ```
    
    Notice that each CISKubeBenchReport is named after a node and is controlled by that node to inherit its life cycle:
    
    ```
    $ kubectl tree node ip-10-250-20-146.eu-central-1.compute.internal -A
    NAMESPACE        NAME                                                                 READY  REASON        AGE  
                    Node/ip-10-250-20-146.eu-central-1.compute.internal                  True   KubeletReady  6h32m
                    ├─CISKubeBenchReport/ip-10-250-20-146.eu-central-1.compute.internal  -                    6h30m
                    ├─CSINode/ip-10-250-20-146.eu-central-1.compute.internal             -                    6h32m
    kube-node-lease └─Lease/ip-10-250-20-146.eu-central-1.compute.internal               -                    6h32m

    ```
    
