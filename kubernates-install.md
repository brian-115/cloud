# Kubernates (K8S) install / config on GCP

Requirement

* gcloud
* kubectl

Document: https://blog.gcp.expert/gke-k8s-pod-network/

## 安裝 kubectl

```
$ sudo yum install -y kubectl
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"13", GitVersion:"v1.13.0", GitCommit:"ddf47ac13c1a9483ea035a79cd7c10005ff21a6d", GitTreeState:"clean", BuildDate:"2018-12-03T21:04:45Z", GoVersion:"go1.11.2", Compiler:"gc", Platform:"linux/amd64"}
The connection to the server localhost:8080 was refused - did you specify the right host or port?

```

## Configure kubectl

## 建立 K8S on GCP

```
[centos@itu9adm-1 ~]$ gcloud config list
[compute]
zone = asia-east1-b
[core]
account = itu9-admin-service-account@wpgcloud-201706.iam.gserviceaccount.com
disable_usage_reporting = True
project = wpgcloud-201706

Your active configuration is: [default]

```

### Create subnet for kubernates

```
$ gcloud compute networks subnets create kubernates \
  --network=gcpvpc --region=asia-east1 \
  --range=10.8.51.0/24 --secondary-range pods01=172.16.0.0/16,services01=172.17.0.0/16 \
  --enable-private-ip-google-access
  
Created [https://www.googleapis.com/compute/v1/projects/wpgcloud-201706/regions/asia-east1/subnetworks/kubernates].
NAME        REGION      NETWORK  RANGE
kubernates  asia-east1  gcpvpc   10.8.51.0/24

```

> *** Cluster.cluster_ipv4_cidr CIDR block size must be no bigger than /9 and no smaller than /19 ***

```
$ gcloud container clusters create "kube-cluster-1" \
  --region "asia-east1" \
  --cluster-version "1.11.3-gke.18" \
  --enable-ip-alias --enable-private-nodes --enable-private-endpoint --enable-master-authorized-networks \
  --master-authorized-networks=10.8.48.0/24 \
  --network gcpvpc --subnetwork kubernates \
  --cluster-secondary-range-name=pods01 --services-secondary-range-name=services01 \
  --master-ipv4-cidr 172.18.0.0/28 \
  --machine-type "g1-small" --image-type "COS" --disk-type "pd-standard" --disk-size "100" \
  --num-nodes "1" --enable-cloud-logging --enable-cloud-monitoring \
  --no-enable-basic-auth \
  --no-issue-client-certificate

WARNING: Starting in 1.12, default node pools in new clusters will have their legacy Compute Engine instance metadata endpoints disabled by default. To create a cluster with legacy instance metadata endpoints disabled in the default node pool, run `clusters create` with the flag `--metadata disable-legacy-endpoints=true`.
This will enable the autorepair feature for nodes. Please see https://cloud.google.com/kubernetes-engine/docs/node-auto-repair for more information on node autorepairs.
WARNING: Starting in Kubernetes v1.10, new clusters will no longer get compute-rw and storage-ro scopes added to what is specified in --scopes (though the latter will remain included in the default --scopes). To use these scopes, add them explicitly to --scopes. To use the new behavior, set container/new_scopes_behavior property (gcloud config set container/new_scopes_behavior true).
Creating cluster kube-cluster-1 in asia-east1...done.
Created [https://container.googleapis.com/v1/projects/wpgcloud-201706/zones/asia-east1/clusters/kube-cluster-1].
To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/asia-east1/kube-cluster-1?project=wpgcloud-201706
kubeconfig entry generated for kube-cluster-1.
NAME            LOCATION    MASTER_VERSION  MASTER_IP   MACHINE_TYPE  NODE_VERSION   NUM_NODES  STATUS
kube-cluster-1  asia-east1  1.11.3-gke.18   172.18.0.2  g1-small      1.11.3-gke.18  3          RUNNING

```

```
$ gcloud container clusters get-credentials kube-cluster-1 --region asia-east1 --project wpgcloud-201706
Fetching cluster endpoint and auth data.
kubeconfig entry generated for kube-cluster-1.

```

```
$ kubectl get nodes
NAME                                            STATUS   ROLES    AGE   VERSION
gke-kube-cluster-1-default-pool-38edd430-n2pk   Ready    <none>   19m   v1.11.3-gke.18
gke-kube-cluster-1-default-pool-8e7d508d-41k6   Ready    <none>   19m   v1.11.3-gke.18
gke-kube-cluster-1-default-pool-db8989f3-df87   Ready    <none>   19m   v1.11.3-gke.18

```

```
$ gcloud compute instances list --filter="name:gke-kube"
NAME                                           ZONE          MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP  STATUS
gke-kube-cluster-1-default-pool-8e7d508d-41k6  asia-east1-a  g1-small                   10.8.51.2                 RUNNING
gke-kube-cluster-1-default-pool-db8989f3-df87  asia-east1-b  g1-small                   10.8.51.4                 RUNNING
gke-kube-cluster-1-default-pool-38edd430-n2pk  asia-east1-c  g1-small                   10.8.51.3                 RUNNING

```

```
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
