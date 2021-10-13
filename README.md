
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

     Send custom resource definitions to the Kubernetes API:

    `kubectl apply -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/crd/vulnerabilityreports.crd.yaml \
    -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/crd/configauditreports.crd.yaml \
    -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/crd/clusterconfigauditreports.crd.yaml \
    -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/crd/ciskubebenchreports.crd.yaml`
  
     Send the following Kubernetes objects definitions to the Kubernetes API:

    `kubectl apply -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/01-starboard-operator.ns.yaml \
    -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/02-starboard-operator.sa.yaml \
    -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/03-starboard-operator.clusterrole.yaml \
    -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/04-starboard-operator.clusterrolebinding.yaml`

     (Optional) Configure Starboard by creating the starboard ConfigMap and the starboard secret in the starboard-operator namespace.
    
    `kubectl apply -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/05-starboard-operator.config.yaml`
    
     Finally, create the starboard-operator Deployment in the starboard-operator namespace to start the operator's pod:
     
    `kubectl apply -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/06-starboard-operator.deployment.yaml`
    
    To confirm that the operator is running, check the number of replicas created by the starboard-operator Deployment in the starboard-operator namespace:
    
    `$ kubectl get deployment -n starboard-operator`
    
    **You can uninstall the operator with the following command:**
    
    `kubectl delete -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/06-starboard-operator.deployment.yaml \
    -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/05-starboard-operator.config.yaml \
    -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/04-starboard-operator.clusterrolebinding.yaml \
    -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/03-starboard-operator.clusterrole.yaml \
    -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/02-starboard-operator.sa.yaml \
    -f https://raw.githubusercontent.com/aquasecurity/starboard/v0.12.0/deploy/static/01-starboard-operator.ns.yaml`

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
