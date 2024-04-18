# rhocp-clusters

1. help configure an existing OpenShift cluster as a *Hub cluster*, whereby it can deploy *Managed* OpenShift clusters with Red Hat Advanced Cluster Management (ACM) for Kubernetes and OpenShift GitOps.Red Hat ACM is responsible for deploying additional OpenShift clusters, typically referred to as *Managed* clusters.
2. contains respective SiteConfig and PolicyGenTemplates responsible for configuring *Managed* OpenShift Clusters to act as Telco DU nodes.  These Telco DU nodes will be configured with necessary platform tuning and configuration, ready for DU onboarding.

> **NOTE:** __All__ cluster lifecycle management is performed by OpenShift Gitops (Argo CD under the covers), including the Argo configuration itself.

## Prerequisites
In order to deploy this document successfully it requires that the following prerequisites are met.  
  - A pull-secret.txt downloaded from [console.redhat.com](https://console.redhat.com/openshift/install/metal/installer-provisioned)
  - A *pre-existing OpenShift installation* running RHOCP version 4.14.2 or greater.
  - OpenShift's Local Storage Operator (LSO) Installed via extra_manifests provision.
  - Required DNS Entries for Hub Cluster.
  - Required DNS Entries for Managed Clusters.


### Patching The Storageclass
RHACM requires the use of a _default_ storageclass. This should already be set to _lso-filesystemclass_. 
```bash
$ oc patch storageclass lso-filesystemclass -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

storageclass.storage.k8s.io/lso-filesystemclass patched
```

### Which Is The Default Storageclass
After the storageclass has been patched you can confirm the patch by running the same `oc get sc` command as earlier.
Reviewing the sample output below you can see that _lso-filesystemclass_ is now marked as `default`.

```bash
$ oc get sc

NAME                            PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
lso-filesystemclass (default)   kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  27m
```

### Appling The Bootstrap Manifests
Once The Storageclass is patched and a default storageclass is defined, it's now time to apply the _bootstrap_ manifests.
The command below once executed will continuously reapply until each of the manifests report _unchanged_.

```bash
$ until oc apply -k /home/redhat/<path to ocp-deployment repo>/bootstrap/overlays/default; do sleep 3; done

clusterrolebinding.rbac.authorization.k8s.io/cluster-admin-for-osgo created
subscription.operators.coreos.com/openshift-gitops-operator created
resource mapping not found for name: "policy-app-project" namespace: "openshift-gitops" from "/home/redhat/<path to ocp-deployment repo>/bootstrap/overlays/default":
no matches for kind "AppProject" in version "argoproj.io/v1alpha1"
ensure CRDs are installed first
... <snipped> ...
appproject.argoproj.io/policy-app-project created
appproject.argoproj.io/ztp-app-project created
applicationset.argoproj.io/cluster created
Warning: resource argocds/openshift-gitops is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by oc apply. oc apply should only be used on resources created declaratively by either oc create --save-config or oc apply. The missing annotation will be patched automatically.
argocd.argoproj.io/openshift-gitops configured
subscription.operators.coreos.com/openshift-gitops-operator unchanged
```
<br /><br />

## Secrets

### Creating A Gitops Secret
Applying the _bootstrap_ manifest triggers the _openshift-gitops_ install.
The _openshift-gitops_ install requires a secret to with the details of the git repo to be used for _openshift-gitops_.  This is required if the git repo is *PRIVATE*.
The command below creates a generic secret called `intel-hub-repo` the contains the `username`, `password`, and `url` to be used with _openshift-gitops_.


```bash
$ oc create -n openshift-gitops secret generic intel-hub-repo \
  --from-literal=username=not-used \
  --from-literal=password=<ORG ACCESS TOKEN> \
  --from-literal=type=git \
  --from-literal=url=https://github.com/jnunyez/rh-ocp.git \

secret/intel-hub-repo created
```
### Labeling A Gitops Secret
Now that a secret has been created, the secret needs to be label as type `repository`.

```bash
$ oc label -n openshift-gitops secret intel-hub-repo argocd.argoproj.io/secret-type=repository

secret/intel-hub-repo labeled
```

### Deleting a gitops secret
If order to modify the previously created secret you will need to delete it and recreate it.
```bash
$ oc delete -n openshift-gitops secret intel-hub-repo
```
<br /><br />

## Validating The Hub Cluster

### Get Openshift-Gitops Applications
It is important to note that blank output doesn't signify an ERROR.
The bootstrap can take a few minutes before you have a useable output.  

```bash
$ oc get applications.argoproj.io -n openshift-gitops
NAME                               SYNC STATUS   HEALTH STATUS
advanced-cluster-management        OutOfSync     Healthy
argo-clusters                      OutOfSync     Healthy
gitops-controller                  Synced        Healthy
topology-aware-lifecycle-manager   Synced        Healthy
```

### RHACM WebConsole
Point you browser to `http://console-openshift-console.apps.<cluster_name>.<cluster_domain>`.
Login with your kube credentials.

### ArgoCD Console
Point your browser to `https://openshift-gitops-server-openshift-gitops.apps.<cluster_name>.<cluster_domain>`.

### Getting The Host/Port For The Bootstrapped Operators
You can also get the URL's from the Hub Cluster itself using `oc get routes -A`.

```bash
$ oc get routes -A
NAMESPACE                            NAME                       HOST/PORT                                                             PATH        SERVICES                   PORT                     TERMINATION            WILDCARD
multicluster-engine                  agent-registration         agent-registration-multicluster-engine.apps.yugo.cars2.lab                        agent-registration         agentregistration        reencrypt/Redirect     None
multicluster-engine                  assisted-image-service     assisted-image-service-multicluster-engine.apps.yugo.cars2.lab                    assisted-image-service     assisted-image-service   reencrypt              None
multicluster-engine                  assisted-service           assisted-service-multicluster-engine.apps.yugo.cars2.lab                          assisted-service           assisted-service         reencrypt              None
multicluster-engine                  cluster-proxy-addon-anp    cluster-proxy-anp.apps.yugo.cars2.lab                                             cluster-proxy-addon-anp    anp-port                 passthrough            None
multicluster-engine                  cluster-proxy-addon-user   cluster-proxy-user.apps.yugo.cars2.lab                                            cluster-proxy-addon-user   user-port                reencrypt/Redirect     None
multicluster-engine                  hcp-cli-download           hcp-cli-download-multicluster-engine.apps.yugo.cars2.lab                          hcp-cli-download           http                     edge/Redirect          None
openshift-authentication             oauth-openshift            oauth-openshift.apps.yugo.cars2.lab                                               oauth-openshift            6443                     passthrough/Redirect   None
openshift-console                    console                    console-openshift-console.apps.yugo.cars2.lab                                     console                    https                    reencrypt/Redirect     None
openshift-console                    downloads                  downloads-openshift-console.apps.yugo.cars2.lab                                   downloads                  http                     edge/Redirect          None
openshift-gitops                     kam                        kam-openshift-gitops.apps.yugo.cars2.lab                                          kam                        8443                     passthrough/None       None
openshift-gitops                     openshift-gitops-server    openshift-gitops-server-openshift-gitops.apps.yugo.cars2.lab                      openshift-gitops-server    https                    passthrough/Redirect   None
openshift-ingress-canary             canary                     canary-openshift-ingress-canary.apps.yugo.cars2.lab                               ingress-canary             8080                     edge/Redirect          None
openshift-monitoring                 alertmanager-main          alertmanager-main-openshift-monitoring.apps.yugo.cars2.lab            /api        alertmanager-main          web                      reencrypt/Redirect     None
openshift-monitoring                 prometheus-k8s             prometheus-k8s-openshift-monitoring.apps.yugo.cars2.lab               /api        prometheus-k8s             web                      reencrypt/Redirect     None
openshift-monitoring                 prometheus-k8s-federate    prometheus-k8s-federate-openshift-monitoring.apps.yugo.cars2.lab      /federate   prometheus-k8s             web                      reencrypt/Redirect     None
openshift-monitoring                 thanos-querier             thanos-querier-openshift-monitoring.apps.yugo.cars2.lab               /api        thanos-querier             web                      reencrypt/Redirect     None
openshift-user-workload-monitoring   federate                   federate-openshift-user-workload-monitoring.apps.yugo.cars2.lab       /federate   prometheus-user-workload   federate                 reencrypt/Redirect     None
openshift-user-workload-monitoring   thanos-ruler               thanos-ruler-openshift-user-workload-monitoring.apps.yugo.cars2.lab   /api        thanos-ruler               web                      reencrypt/Redirect     None
```


### Check that the ztp plugin has been loaded. 

ZTP requires that a plugin is loaded into the gitops operator this should be done by the bootstrap manifests and can be checked by running the following command.

```bash
$ oc get argocds.argoproj.io -n openshift-gitops -oyaml | grep ztp
        image: registry.redhat.io/openshift4/ztp-site-generate-rhel8:v4.14
```
<br /><br />

## Managed Clusters


### Apply The Hub Prerequisites
The first step in getting your Managed Cluster deployed is to apply the `00-hub-prereqs` manifests.
This will create the namespace and BMC secret for your Managed Cluster.

```bash
$ oc apply -k /<path to ocp-deployment repo>/clusters/<CLUSTER_NAME>.<CLUSTER_DOMAIN>/00-hub-prereqs
namespace/bevo2 created

secret/cars2-sno-bevo2-cars2-test-bmc-secret created
```

### Add Pull Secret to Secrets
We must also create another secret to store the pull secret downloaded from console.redhat.com.
This pull-secret is used by the _Management Cluster_ to pull images from the registry.
This can same pull secret used to deploy the _Hub Cluster_ but does not have to be.

```bash
$ oc create secret generic assisted-deployment-pull-secret -n tse227 --from-file=.dockerconfigjson=<path to pull secret> --type=kubernetes.io/dockerconfigjson

secret/assisted-deployment-pull-secret created
```

### Apply the 01-hub-apps Manifests
Next we will apply the gitops application which will sync the siteconfig to the Hub Cluster.

```bash
$ oc apply -k /home/redhat/<path to ocp-deployment repo>/clusters/bevo2.cars2.lab/01-hub-apps

application.argoproj.io/cars-clusters-bevo2 created
```

### Apply ZTP Policy Hub Apps
At this point we need to configure the Managed Cluster by applying the `hub-apps` policy templates to the Hub Cluster.

```bash
$ oc apply -k /home/redhat/<path to ocp-deployment repo>/clusters/ztp-policies/hub-apps

namespace/policies-sub created
application.argoproj.io/common-policies created
```

### Query Installation Progress
At this point your Managed Cluster is being installed.
We can monitor the progress of the installation from the cli with the following `watch` command.

```bash
$ watch 'oc get bmh,infraenv,agentclusterinstalls,managedclusters,agents -A; echo Progress $(oc get agentclusterinstalls -n bevo2 bevo2 -o json | jq .status.progress.totalPercentage) %'

NAMESPACE               NAME                                                  STATE          CONSUMER              ONLINE   ERROR   AGE
bevo2                   r750-2.bevo2.cars2.lab   provisioned                          true             8d
openshift-machine-api   supervisor1              unmanaged     aspid-55rd2-master-0   true             23d
openshift-machine-api   supervisor2              unmanaged     aspid-55rd2-master-1   true             23d
openshift-machine-api   supervisor3              unmanaged     aspid-55rd2-master-2   true             23d

NAMESPACE   NAME    ISO CREATED AT
bevo2       bevo2   2024-04-10T11:22:02Z

NAMESPACE   NAME    CLUSTER   STATE
bevo2       bevo2   bevo2     adding-hosts

NAMESPACE   NAME                                                              HUB ACCEPTED   MANAGED CLUSTER URLS               JOINED   AVAILABLE   AGE
            managedcluster.cluster.open-cluster-management.io/bevo2           true           https://api.bevo2.cars2.lab:6443   True     True        24h
            managedcluster.cluster.open-cluster-management.io/local-cluster   true           https://api.aspid.cars2.lab:6443   True     True        23d

NAMESPACE   NAME                                                                    CLUSTER   APPROVED   ROLE     STAGE
bevo2       agent.agent-install.openshift.io/96323eb2-b697-abba-b8ab-19a661511922   bevo2     true       master   Done
Progress 100 %
```


> NOTE: The progress bar in this step only covers installation of the Management Cluster.
It does not include configuration of the Management Cluster. 


### The State of the Agents
Using the command below you can check the state of the agents in connection with your Managed Cluster.
```bash
$ oc get agents.agent-install.openshift.io -n bevo2  -o=jsonpath='{range .items[*]}{"\n"}{.spec.clusterDeploymentName.name}{"\n"}{.status.inventory.hostname}{"\n"}{range .status.conditions[*]}{.type}{"\t"}{.message}{"\n"}{end}'

bevo2
r750-2.bevo2.cars2.lab
SpecSynced	The Spec has been successfully applied
Connected	The agent's connection to the installation service is unimpaired
RequirementsMet	The agent installation stopped
Validated	The agent's validations are passing
Installed	The installation has completed: Done
Bound	The agent is bound to a cluster deployment
```

### Monitoring The Compliance Of Applied Policies
You can run the command below if you want to see how the Managed Cluster is progressing towards compliance.

```bash
$ oc get policies.policy.open-cluster-management.io -A
NAMESPACE    NAME                                                           REMEDIATION ACTION   COMPLIANCE STATE   AGE
bevo2        ztp-common.common-du-414-config-policy                         inform               Compliant          24h
bevo2        ztp-common.common-du-414-log-forwarder-policy                  inform               Compliant          24h
bevo2        ztp-common.common-du-414-subscriptions-policy                  inform               Compliant          24h
bevo2        ztp-group.group-dellr750-vse6-config-policy                    inform               Compliant          24h
bevo2        ztp-group.group-dellr750-vse6-fec-policy                       inform               Compliant          24h
bevo2        ztp-group.group-dellr750-vse6-sriov-netd-policy-ens2f0         inform               Compliant          24h
bevo2        ztp-group.group-dellr750-vse6-sriov-netd-policy-ens5f0         inform               Compliant          24h
bevo2        ztp-group.group-dellr750-vse6-sriov-netd-policy-ens5f1         inform               Compliant          24h
bevo2        ztp-group.group-dellr750-vse6-sriov-network-policy-bh          inform               Compliant          24h
bevo2        ztp-group.group-dellr750-vse6-sriov-network-policy-mh-f1c      inform               Compliant          24h
bevo2        ztp-group.group-dellr750-vse6-sriov-network-policy-mh-f1u      inform               Compliant          24h
bevo2        ztp-group.group-dellr750-vse6-sriov-network-policy-vran-fh     inform               Compliant          24h
bevo2        ztp-group.group-dellr750-vse6-sriov-network-policy-vran-fh-c   inform               Compliant          24h
bevo2        ztp-group.group-dellr750-vse6-sriov-network-policy-vran-mgmt   inform               Compliant          24h
bevo2        ztp-group.group-dellr750-vse6-sriov-vfio-policy-ens2f0         inform               Compliant          24h
bevo2        ztp-group.group-dellr750-vse6-sriov-vfio-policy-ens5f0         inform               Compliant          24h
bevo2        ztp-group.group-dellr750-vse6-sriov-vfio-policy-ens5f1         inform               Compliant          24h
bevo2        ztp-group.group-dellr750-vse6-subscription-policy              inform               Compliant          24h
ztp-common   common-du-414-config-policy                                    inform               Compliant          21d
ztp-common   common-du-414-log-forwarder-policy                             inform               Compliant          6d19h
ztp-common   common-du-414-subscriptions-policy                             inform               Compliant          21d
ztp-group    group-dellr750-vse6-config-policy                              inform               Compliant          21d
ztp-group    group-dellr750-vse6-fec-policy                                 inform               Compliant          21d
ztp-group    group-dellr750-vse6-sriov-netd-policy-ens2f0                   inform               Compliant          21d
ztp-group    group-dellr750-vse6-sriov-netd-policy-ens5f0                   inform               Compliant          21d
ztp-group    group-dellr750-vse6-sriov-netd-policy-ens5f1                   inform               Compliant          21d
ztp-group    group-dellr750-vse6-sriov-network-policy-bh                    inform               Compliant          21d
ztp-group    group-dellr750-vse6-sriov-network-policy-mh-f1c                inform               Compliant          21d
ztp-group    group-dellr750-vse6-sriov-network-policy-mh-f1u                inform               Compliant          21d
ztp-group    group-dellr750-vse6-sriov-network-policy-vran-fh               inform               Compliant          21d
ztp-group    group-dellr750-vse6-sriov-network-policy-vran-fh-c             inform               Compliant          21d
ztp-group    group-dellr750-vse6-sriov-network-policy-vran-mgmt             inform               Compliant          21d
ztp-group    group-dellr750-vse6-sriov-vfio-policy-ens2f0                   inform               Compliant          20d
ztp-group    group-dellr750-vse6-sriov-vfio-policy-ens5f0                   inform               Compliant          21d
ztp-group    group-dellr750-vse6-sriov-vfio-policy-ens5f1                   inform               Compliant          21d
ztp-group    group-dellr750-vse6-subscription-policy                        inform               Compliant          21d
ztp-group    group-dellr750-vse6-validator-du-policy                        inform                                  21d
```


### Using The Managed Cluster
You can use the bash function below as an easy of getting the kubeconfig and kubeadmin for your Managed Cluster

```bash
get_kubeconfig () {
    oc get secret -n $1 $1-admin-kubeconfig -o json | jq -r '.data.kubeconfig' | base64 -d > ~/kubeconfig-$1
    oc get secret -n $1 $1-admin-password -o json | jq -r '.data.password' | base64 -d > ~/kubeadmin-$1
}
```
>Usage: `$ get_kubeconfig <CLUSTER_NAME>`

