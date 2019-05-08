# GKE Config and Deploy

TOC
* Deploy a POD
* Deploy a workload
* Expose Service
* Logging and Monitoring

## Deploy a POD

### Create a Pod YAML file

```sh
$ kubectl run --dry-run wpg-nginx --image=nginx --restart=Never -o yaml > wpg-nginx-pod.yaml
```

```sh
$ cat wpg-nginx-pod.yaml
```

### Deploy Pod

```sh
$ kubectl create -f wpg-nginx-pod.yaml
```

### Inspect Pod

Show Pods

```sh
$ kubectl get pods
```

Show Pods with more detail

```sh
$ kubectl get pods -o wide
$ kubectl describe pods wpg-nginx
```

Show Pods resource status

```sh
$ kubectl top pod
```

> Browse Pod status in GKE in cloud console

### Access Pod

```sh
$ kubectl port-forward nginx 8080:80
$ curl http://localhost:8080
```

## Monitoring and Logging

### Stackdriver Logging

> Browse logs in Stackdriver Logging of cloud console

| 資源類型 | 顯示名稱 |
| ---- | ---- |
| k8s_cluster |	Kubernetes 叢集 |
| gke_cluster | GKE 叢集作業 |
| gke_container | GKE 容器 |
| gke_nodepool | GKE 節點集區作業 |

Watch log of Pod

```sh
$ kubectl logs nginx
```
```sh
$ kubectl logs -f nginx
```

### Stackdriver Monitoring

