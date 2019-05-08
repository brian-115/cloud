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
$ gcloud beta container clusters create "gke-cluster-1" \
  --region "asia-east1" --no-enable-basic-auth \
  --cluster-version "1.12.7-gke.10" --machine-type "g1-small" --image-type "COS" \
  --disk-type "pd-standard" --disk-size "100" \
  --num-nodes "1" --enable-stackdriver-kubernetes \
  --enable-private-nodes --master-ipv4-cidr "172.18.0.0/28" --enable-ip-alias \
  --network "gcpvpc" --subnetwork "kubernates" \
  --cluster-secondary-range-name "pods01" --services-secondary-range-name "services01" \
  --enable-master-authorized-networks --master-authorized-networks 0.0.0.0/0 \
  --enable-autoupgrade --enable-autorepair --maintenance-window "13:00"



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
$ gcloud container clusters get-credentials gke-cluster-1 --region asia-east1 
Fetching cluster endpoint and auth data.
kubeconfig entry generated for kube-cluster-1.
```

Show `kubectl` current context

```sh
$ kubectl config current-context
gke_itu9-poc-20190430_asia-east1_kube-cluster-1
```

Show cluster infomation

```
$ kubectl cluster-info
```




## Relative command for kubectl

List nodes : `kubectl get nodes`

```sh
$ kubectl get nodes
NAME                                            STATUS   ROLES    AGE   VERSION
gke-kube-cluster-1-default-pool-38edd430-n2pk   Ready    <none>   19m   v1.11.3-gke.18
gke-kube-cluster-1-default-pool-8e7d508d-41k6   Ready    <none>   19m   v1.11.3-gke.18
gke-kube-cluster-1-default-pool-db8989f3-df87   Ready    <none>   19m   v1.11.3-gke.18
```

Review GCEs

```sh
$ gcloud compute instances list --filter="name:gke-kube"
NAME                                           ZONE          MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP  STATUS
gke-kube-cluster-1-default-pool-8e7d508d-41k6  asia-east1-a  g1-small                   10.8.51.2                 RUNNING
gke-kube-cluster-1-default-pool-db8989f3-df87  asia-east1-b  g1-small                   10.8.51.4                 RUNNING
gke-kube-cluster-1-default-pool-38edd430-n2pk  asia-east1-c  g1-small                   10.8.51.3                 RUNNING
```

Review Firewall rules

```sh
$ gcloud container clusters describe kube-cluster-1 --region asia-east1

$ gcloud compute firewall-rules list \
  --filter 'name~^gke-kube-cluster-1' \
  --format 'table(
        name,
        network,
        direction,
        sourceRanges.list():label=SRC_RANGES,
        allowed[].map().firewall_rule().list():label=ALLOW,
        targetTags.list():label=TARGET_TAGS
    )'
NAME                                NETWORK  DIRECTION  SRC_RANGES     ALLOW                         TARGET_TAGS
gke-kube-cluster-1-a6b1e23f-all     gcpvpc   INGRESS    172.16.0.0/16  tcp,udp,icmp,esp,ah,sctp      gke-kube-cluster-1-a6b1e23f-node
gke-kube-cluster-1-a6b1e23f-master  gcpvpc   INGRESS    172.18.0.0/28  tcp:10250,tcp:443             gke-kube-cluster-1-a6b1e23f-node
gke-kube-cluster-1-a6b1e23f-vms     gcpvpc   INGRESS    10.8.51.0/24   udp:1-65535,icmp,tcp:1-65535  gke-kube-cluster-1-a6b1e23f-node
```
