# GKE Quick Start

Rrequirement

* gcloud
* kubectl

Document: https://cloud.google.com/sdk/install

Enable Cloud APIs

```
gcloud services enable container.googleapis.com
gcloud services enable cloudbuild.googleapis.com
gcloud services enable containerregistry.googleapis.com
gcloud services enable logging.googleapis.com
gcloud services enable monitoring.googleapis.com
```

**TOC**

* Create a Cluster
* Create a Pod
* Deploy a workload
* Expose Service
* Logging and Monitoring

## Create a cluster

Single Zone Cluster

```
$ gcloud beta container clusters create gke-zone-demo \
  --zone asia-east1-b \
  --enable-ip-alias \
  --enable-stackdriver-kubernetes
```

Regional Cluster

```
$ gcloud beta container clusters create gke-region-demo \
  --region asia-east1 \
  --enable-ip-alias \
  --enable-stackdriver-kubernetes
```

Using `--enable-private-nodes` and `--enable-master-authorized-networks` flags to create private cluster.

![](https://storage.googleapis.com/gweb-cloudblog-publish/images/image2mhii.max-700x700.PNG)

## Access cluster and browse resources

Setup `kubectl` credentials

```
$ gcloud container clusters get-credentials gke-zone-demo \
  --zone asia-east1-b
```

Show `kubectl`'s current context

```
$ kubectl config current-context
```

Show cluster infomation

```
$ kubectl cluster-info
```

### Nodes

Show Nodes of cluster

```
$ kubectl get nodes
```

Show Nodes of cluster with more detail

```
$ kubectl get nodes -o wide
```

```
$ kubectl describe nodes [NODE_NAME]
```

Show Nodes resource status

```
$ kubectl top nodes
```

> Browse cluster status in GKE in cloud console

## Deploy a pod

![](https://d33wubrfki0l68.cloudfront.net/aecab1f649bc640ebef1f05581bfcc91a48038c4/728d6/images/docs/pod.svg)

Create a Pod YAML file

```
$ kubectl run --dry-run nginx --image=nginx --restart=Never -o yaml > pod.yaml
```

```
$ cat pod.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

Deploy Pod

```
$ kubectl create -f pod.yaml
```

### Inspect Pod

Show Pods

```
$ kubectl get pods
```

Show Pods with more detail

```
$ kubectl get pods -o wide
```

```
$ kubectl describe pods nginx
```

Show Pods resource status

```
$ kubectl top pod
```

> Browse Pod status in GKE in cloud console

### Access Pod

```
$ kubectl port-forward nginx 8080:80
```

```
$ curl http://localhost:8080
```

## Monitoring and Logging

### Stackdriver Logging

> Browse logs in Sstackdriver Logging of cloud console

| 資源類型 | 顯示名稱 |
| ---- | ---- |
| k8s_cluster |	Kubernetes 叢集 |
| gke_cluster | GKE 叢集作業 |
| gke_container | GKE 容器 |
| gke_nodepool | GKE 節點集區作業 |

Watch log of Pod

```
$ kubectl logs nginx
```

```
$ kubectl logs -f nginx
```

### Stackdriver Monitoring

![](https://storage.googleapis.com/gweb-cloudblog-publish/original_images/Stackdriver-Kubernetes-Monitoring-prometheus8qyj.GIF)

## Configuration

### Environment and ConfigMap

Create a ConfigMap YAML file

```
$ kubectl create --dry-run configmap dummy-config \
  --from-literal=foo=bar -o yaml > cm.yaml
```

```
$ cat cm.yaml
```

```
$ kubectl create -f cm.yaml
```

Show ConfigMaps

```
$ kubectl get configmaps
```

**Use ConfigMaps as environment variable to Conatiner**

Edit `pod.yaml`

```
spec:
  containers:
   env:
    - name: FOO
      valueFrom:
        configMapKeyRef:
          name: dummy-config
          key: foo
```

```
$ kubectl delete pod nginx
$ kubectl apply -f pod.yaml
```

Show environment variables of container

```
$ kubectl exec -it nginx -- env
```

**Use ConfigMaps as Directory to Conatiner**

Generate file

```
$ echo "Hello world" > welcome.txt
```

Create ConfigMaps YAML file

```
$ kubectl create --dry-run configmap welcome-config \
  --from-file=welcome.txt -o yaml > welcome.yaml
```

```
$ kubectl create -f welcome.yaml
```

Edit `pod.yaml`

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
      - name: welcome-volume
        mountPath: /etc/config
  volumes:
    - name: welcome-volume
      configMap:
        name: welcome-config
```

```
$ kubectl delete pod nginx
$ kubectl apply -f pod.yaml
```

```
$ kubectl exec -it nginx -- ls /etc/config
$ kubectl exec -it nginx -- cat /etc/config/welcome.txt
```

Document: https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/

### Liveness and Readiness

**Liveness Demo**

Edit `pod.yaml`

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 3
```

Redeploy pod

```
$ kubectl delete pod nginx
$ kubectl apply -f pod.yaml
```

Watch Pod's status in different window

```
$ watch kubectl get pods
```

Delete index of Pod

```
$ kubectl exec -it nginx -- rm /usr/share/nginx/html/index.html
```

**Readiness Demo**

Edit `pod.yaml`

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 3
```

Redeploy pod

```
$ kubectl delete pod nginx
$ kubectl apply -f pod.yaml
```

Watch Pod's status in different window

```
$ watch kubectl get pods
```

Delete index of Pod

```
$ kubectl exec -it nginx -- rm /usr/share/nginx/html/index.html
```

Document: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/

### Resource limit

> OOM will restart container.

Edit `pod.yaml`

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    resources:
      requests:
        memory: "5Mi"
        cpu: "200m"
      limits:
        memory: "6Mi"
        cpu: "250m"
```

Redeploy pod

```
$ kubectl delete pod nginx
$ kubectl apply -f pod.yaml
```

Watch Pod's status Watch Pod's status in different window

```
$ watch kubectl get pods
```

```
$ watch kubectl top pods
```

Access Pod

```
$ kubectl exec -it nginx -- bash
```

Generate memory stress

```
> BYTES=$((1024*1024*100))
> while(true); do bash -c "cat <(yes | tr \\n x | head -c $BYTES) >> /dev/shm/a;" sleep 1;done
```

* Kubernetes Document: https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/
* GKE Best Practice: https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-resource-requests-and-limits
* GKE Reserved resources: https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-architecture#node_allocatable

## Deploy a web application

### Build Image

Get sample project

```
$ git clone https://github.com/GoogleCloudPlatform/kubernetes-engine-samples
$ cd kubernetes-engine-samples/hello-app
```

Set the project id(optional)
```
$ export PROJECT_ID="$(gcloud config get-value project -q)"
```

Build container image

**Use Docker**

```
$ gcloud auth configure-docker
```

```
$ docker push gcr.io/${PROJECT_ID}/hello-app:v1
```

**Use CloudBuild**

```
$ gcloud builds submit --tag="gcr.io/${PROJECT_ID}/hello-app:v1" .
```

> Browse Google Container Registry and Cloud Build in cloud console

Reference: https://cloud.google.com/kubernetes-engine/docs/tutorials/gitops-cloud-build?hl=en

### Deploy Web Application

```
$ kubectl run --dry-run hello-web --image=gcr.io/${PROJECT_ID}/hello-app:v1 \
  --port 8080 -o yaml > deploy.yaml
```

```
$ cat deploy.yaml
```

```
$ kubectl create -f deploy.yaml
```

Show deployments

```
$ kubectl get deployments
```

Show Deployments with more detail

```
$ kubectl get deployments -o wide
```

```
$ kubectl describe deployments hello-web
```

Show Pods

```
$ kubectl get pods
```

**Scale Pods**

Edit `deploy.yaml`

```
spec:
   replicas: 2
```

```
$ kubectl apply -f deploy.yaml
```

Show Pods

```
$ kubectl get pods
```

> Browse cluster status in GKE in cloud console

Reference: https://github.com/GoogleCloudPlatform/kubernetes-engine-samples

Hello Application example: https://github.com/GoogleCloudPlatform/kubernetes-engine-samples/tree/master/hello-app

### Access in cluster

Show Pod ips

```
$ kubectl get pods -o wide
```

Create a test client pod

```
$ kubectl run busybox --image=busybox:1.28 --restart=Never --command -- sleep 3600
```

Remote access test client

```
$ kubectl exec -it busybox -- wget -qO- http://[POD_IP]
```

## Service

![](https://d33wubrfki0l68.cloudfront.net/27b2978647a8d7bdc2a96b213f0c0d3242ef9ce0/e8c9b/images/docs/services-iptables-overview.svg)

```
$ kubectl expose --dry-run deployment hello-web --port 80 --target-port 8080 -o yaml > svc.yaml
```

```
$ cat svc.yaml
```

```
$ kubectl create -f svc.yaml
```

Show Services

```
$ kubectl get services
```

Remote access test client

```
$ kubectl exec -it busybox -- wget -qO- http://[SERVICE_CLUSTER_IP]
```

```
$ kubectl exec -it busybox -- nslookup hello-web
```

```
> kubectl exec -it busybox -- wget -qO- http://hello-web
```

The dns record format:

> my-svc.my-namespace.svc.cluster.local

Port forward

```
$ kubectl port-forward services/hello-web 8080:80
```

Document: https://kubernetes.io/docs/concepts/services-networking/service/

### L4 Load Balancer

Edit `svc.yaml`

```
spec:
  type: LoadBalancer
```

```
$ kubectl apply -f svc.yaml
```

```
$ kubectl get services
```

> Test the `EXTERNAL-IP`

> Browse Load Balancer in cloud console

### L7 Load Balancer

Edit `svc.yaml`

```
spec:
  type: NodePort
```

```
$ kubectl apply -f svc.yaml
```

Edit `ing.yaml`

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: hello-web-ingress
spec:
  rules:
  - http:
      paths:
      - path: /*
        backend:
          serviceName: hello-web
          servicePort: 80
```

```
$ kubectl create -f ing.yaml
```

> Wait about 5 minutes...

Show Ingress

```
$ kubectl get ingress
```

Show Pods with more detail

```
$ kubectl describe ingress hello-web-ingress
```

> Browse Load Balancer in cloud console

> Test the `EXTERNAL-IP`

### Set SSL certificate

Create a TLS certificate

```
$ DOMAIN=hello-web.dev
$ openssl genrsa -out $DOMAIN.key 2048
$ openssl req -new -key $DOMAIN.key -out $DOMAIN.csr \
  -subj "/CN=$DOMAIN"
$ openssl x509 -req -days 365 -in $DOMAIN.csr -signkey $DOMAIN.key \
    -out $DOMAIN.crt
```

Upload certificate to Project

```
$ gcloud compute ssl-certificates create hello-web \
  --certificate $DOMAIN.crt \
  --private-key $DOMAIN.key
```

> Browse Load Balancer in cloud console

Edit `ing.yaml`

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: hello-web-ingress
  annotations:
    ingress.gcp.kubernetes.io/pre-shared-cert: "hello-web"
spec:
  rules:
  - host: hello-web.dev
    http:
      paths:
      - path: /*
        backend:
          serviceName: hello-web
          servicePort: 80
```

> Wait about 5 minutes...

```
$ sudo vi /etc/hosts
```

```
$ curl -k https://hello-web.dev
```

Reference: https://cloud.google.com/kubernetes-engine/docs/how-to/ingress-multi-ssl

## Scale & Update

Scale hello-web pods

```
$ kubectl scale deployments hello-web --replicas=5
```

Check

```
$ kubectl get pods
```

**Update container image**

Edit `main.go`

```
  fmt.Fprintf(w, "Version: 2.0.0\n")
```

Build v2 version

```
$ gcloud builds submit --tag="gcr.io/${PROJECT_ID}/hello-app:v2" .
```

Edit `deploy.yaml`

```
    spec:
      containers:
      - image: gcr.io/project-test-share/hello-app:v2
```

```
$ kubectl apply -f deploy.yaml
```

Check update progress

```
$ kubectl rollout status deployments hello-web
```

Check deployed versions

```
$ kubectl rollout history deployments hello-web
```

Rollback

```
$ kubectl rollout undo deployments hello-web
```

## Clean Resources

```
$ kubectl delete -f .
```

```
$ gcloud container clusters delete gke-zone-demo \
  --zone asia-east1-b
```

```
$ gcloud compute ssl-certificates delete hello-web
```

```
$ gcloud container images delete gcr.io/${PROJECT_ID}/hello-app:v1
$ gcloud container images delete gcr.io/${PROJECT_ID}/hello-app:v2
```

# Other References

* Kubernetes Document: https://kubernetes.io/
* Google Kubernetes Engine Document: https://cloud.google.com/kubernetes-engine/docs/
* Kubectl cheat sheet: https://kubernetes.io/docs/reference/kubectl/cheatsheet/
* Kubens: https://github.com/ahmetb/kubectx/blob/master/kubens
* Minikube: https://github.com/kubernetes/minikube
