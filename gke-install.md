# GKE install and config

Requirement

* gcloud
* kubectl
* Cloud Shell (use for this document)

Document: https://blog.gcp.expert/gke-k8s-pod-network/

Enable Cloud APIs (optional)

```sh
gcloud services enable container.googleapis.com
gcloud services enable cloudbuild.googleapis.com
gcloud services enable containerregistry.googleapis.com
gcloud services enable logging.googleapis.com
gcloud services enable monitoring.googleapis.com
```

## Install GKE

Check current configure `gcloud`

```sh
$ gcloud config list
[component_manager]
disable_update_check = True
[compute]
gce_metadata_read_timeout_sec = 5
[core]
account = brian.chang@wpgholdings.com
disable_usage_reporting = False
project = itu9-poc-20190430
[metrics]
environment = devshell

Your active configuration is: [cloudshell-49]
```

Create VPC (optional)

```sh
$ gcloud compute networks create gcpvpc --subnet-mode=custom

Created [https://www.googleapis.com/compute/v1/projects/itu9-poc-20190430/global/networks/gcpvpc].
NAME    SUBNET_MODE  BGP_ROUTING_MODE  IPV4_RANGE  GATEWAY_IPV4
gcpvpc  CUSTOM       REGIONAL
```

Create subnet for kubernates (optional)

```sh
$ gcloud compute networks subnets create kubernates \
  --network=gcpvpc --region=asia-east1 \
  --range=10.8.51.0/24 --secondary-range pods01=172.16.0.0/16,services01=172.17.0.0/16 \
  --enable-private-ip-google-access
  
Created [https://www.googleapis.com/compute/v1/projects/itu9-poc-20190430/regions/asia-east1/subnetworks/kubernates].
NAME        REGION      NETWORK  RANGE
kubernates  asia-east1  gcpvpc   10.8.51.0/24
```

> *** Cluster.cluster_ipv4_cidr CIDR block size must be no bigger than /9 and no smaller than /19 ***

Get GKE cluster version

```sh
$ gcloud container get-server-config --region asia-east1
```

Create cluster for kubernates

```sh
$ gcloud beta container clusters create "wpg-prod-01" \
  --region "asia-east1" --no-enable-basic-auth \
  --cluster-version "1.12.7-gke.10" --machine-type "g1-small" --image-type "COS" \
  --disk-type "pd-standard" --disk-size "100" \
  --num-nodes "1" --enable-stackdriver-kubernetes \
  --enable-private-nodes --master-ipv4-cidr "172.18.0.0/28" --enable-ip-alias \
  --network "gcpvpc" --subnetwork "kubernates" \
  --cluster-secondary-range-name "pods01" --services-secondary-range-name "services01" \
  --enable-master-authorized-networks --master-authorized-networks 0.0.0.0/0 \
  --enable-autoupgrade --enable-autorepair --maintenance-window "13:00"

WARNING: Starting in 1.12, new clusters will not have a client certificate issued. You can manually enable (or disable) the issuance of the client certificate using the `--[no-]issue-client-certificate` flag.
WARNING: Starting in 1.12, default node pools in new clusters will have their legacy Compute Engine instance metadata endpoints disabled by default. To create a cluster with legacy instance metadata endpoints disabled in the default node pool, run `clusters create` with the flag `--metadata disable-legacy-endpoints=true`.
WARNING: The Pod address range limits the maximum size of the cluster. Please refer to https://cloud.google.com/kubernetes-engine/docs/how-to/flexible-pod-cidr to learn how to optimize IP address allocation.
This will enable the autorepair feature for nodes. Please see https://cloud.google.com/kubernetes-engine/docs/node-auto-repair for more information on node autorepairs.
Creating cluster cluster-1 in asia-east1... Cluster is being health-checked (master is healthy)...done.
Created [https://container.googleapis.com/v1beta1/projects/itu9-poc-20190430/zones/asia-east1/clusters/wpg-prod-01].
To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/asia-east1/wpg-prod-01?project=itu9-poc-20190430
kubeconfig entry generated for wpg-prod-01.
NAME           LOCATION    MASTER_VERSION  MASTER_IP      MACHINE_TYPE  NODE_VERSION   NUM_NODES  STATUS
wpg-prod-01    asia-east1  1.12.7-gke.10   35.229.186.85  g1-small      1.12.7-gke.10  3          RUNNING
```

> `--enable-private-endpoint` will cause `kubectl` cannot access master cluster from Cloud Shell.
> https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters#cloud_shell

## Install kubectl (optional)

```sh
$ sudo yum install -y kubectl
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"13", GitVersion:"v1.13.0", GitCommit:"ddf47ac13c1a9483ea035a79cd7c10005ff21a6d", GitTreeState:"clean", BuildDate:"2018-12-03T21:04:45Z", GoVersion:"go1.11.2", Compiler:"gc", Platform:"linux/amd64"}
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

## Access cluster and browse resources

Remove kubectl config (option)

```sh
$ rm -rf ~/.kube
```

Fetch credentials for a running cluster

```sh
$ gcloud container clusters get-credentials wpg-prod-01 --region asia-east1 
Fetching cluster endpoint and auth data.
kubeconfig entry generated for kube-cluster-1.
```

Show `kubectl` current context

```sh
$ kubectl config current-context
gke_itu9-poc-20190430_asia-east1_wpg-prod-01
```

Show cluster infomation

```
$ kubectl cluster-info

Kubernetes master is running at https://35.229.186.85
GLBCDefaultBackend is running at https://35.229.186.85/api/v1/namespaces/kube-system/services/default-http-backend:http/proxy
Heapster is running at https://35.229.186.85/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at https://35.229.186.85/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://35.229.186.85/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Show Nodes of cluster

```sh
$ kubectl get nodes
NAME                                           STATUS   ROLES    AGE   VERSION
gke-wpg-prod-01-default-pool-2c6d3e09-jvn6   Ready    <none>   11m   v1.12.7-gke.10
gke-wpg-prod-01-default-pool-45f8fc02-f21p   Ready    <none>   11m   v1.12.7-gke.10
gke-wpg-prod-01-default-pool-63239796-q3lf   Ready    <none>   11m   v1.12.7-gke.10
```

Show Nodes of cluster with more detail

```sh
$ kubectl get nodes -o wide
NAME                                           STATUS   ROLES    AGE   VERSION          INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                             KERNEL-VERSION   CONTAINER-RUNTIME
gke-wpg-prod-01-default-pool-2c6d3e09-jvn6     Ready    <none>   12m   v1.12.7-gke.10   10.8.51.10                  Container-Optimized OS from Google   4.14.106+        docker://17.3.2
gke-wpg-prod-01-default-pool-45f8fc02-f21p     Ready    <none>   12m   v1.12.7-gke.10   10.8.51.9                   Container-Optimized OS from Google   4.14.106+        docker://17.3.2
gke-wpg-prod-01-default-pool-63239796-q3lf     Ready    <none>   12m   v1.12.7-gke.10   10.8.51.11                  Container-Optimized OS from Google   4.14.106+        docker://17.3.2
```

```sh
$ kubectl describe nodes [NODE_NAME]
```

Show Nodes resource status

```sh
$ kubectl top nodes
```

Review GCEs

```sh
$ gcloud compute instances list --filter="name:gke-"
NAME                                          ZONE          MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP  STATUS
gke-wpg-prod-01-default-pool-63239796-q3lf    asia-east1-a  g1-small                   10.8.51.11                RUNNING
gke-wpg-prod-01-default-pool-45f8fc02-f21p    asia-east1-b  g1-small                   10.8.51.9                 RUNNING
gke-wpg-prod-01-default-pool-2c6d3e09-jvn6    asia-east1-c  g1-small                   10.8.51.10                RUNNING
```

Review Firewall rules

```sh
$ gcloud container clusters describe wpg-prod-01 --region asia-east1

$ gcloud compute firewall-rules list \
  --filter 'name~^gke-wpg-prod-01' \
  --format 'table(
        name,
        network,
        direction,
        sourceRanges.list():label=SRC_RANGES,
        allowed[].map().firewall_rule().list():label=ALLOW,
        targetTags.list():label=TARGET_TAGS
    )'
NAME                               NETWORK  DIRECTION  SRC_RANGES     ALLOW                         TARGET_TAGS
gke-wpg-prod-01-df7d0a5e-all       gcpvpc   INGRESS    172.16.0.0/16  ah,sctp,tcp,udp,icmp,esp      gke-wpg-prod-01-df7d0a5e-node
gke-wpg-prod-01-df7d0a5e-master    gcpvpc   INGRESS    172.18.0.0/28  tcp:10250,tcp:443             gke-wpg-prod-01-df7d0a5e-node
gke-wpg-prod-01-df7d0a5e-vms       gcpvpc   INGRESS    10.8.51.0/24   tcp:1-65535,udp:1-65535,icmp  gke-wpg-prod-01-df7d0a5e-node
```
