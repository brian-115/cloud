# GKE Config POD

POD 為 K8S 最小部署單位, 由一至多個容器所組成, 使其較接近於一般使用的 VM.

TOC
* Create a POD
* View information of POD


## Create a POD

Create a YAML file of POD

```sh
$ kubectl run --dry-run wpg-nginx --image=nginx --restart=Never -o yaml > wpg-nginx-pod.yaml
```

```sh
$ cat wpg-nginx-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: wpg-nginx
  name: wpg-nginx
spec:
  containers:
  - image: nginx
    name: wpg-nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

Create a POD

```sh
$ kubectl create -f wpg-nginx-pod.yaml
```

## View PODs information

List Pods

```sh
$ kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
wpg-nginx                  1/1     Running   0          7s

$ kubectl get pods -o wide
NAME                       READY   STATUS    RESTARTS   AGE   IP           NODE                                           NOMINATED NODE
wpg-nginx                  1/1     Running   0          53s   172.16.0.6   gke-gke-cluster-1-default-pool-45f8fc02-f21p   <none>
```

View POD in detail

```sh
$ kubectl describe pods wpg-nginx
...
```

Show Pods resource status

```sh
$ kubectl top pod
NAME                       CPU(cores)   MEMORY(bytes)
wpg-nginx                  0m           2Mi

$ watch kubectl top pod
```

## Access a POD

Execute command on POD

```sh
$ kubectl exec -it wpg-nginx -- sh
# ...
```

Port Forward to client side

```sh
$ kubectl port-forward nginx 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Handling connection for 8080
```

Create another Cloud Shell Session for test

```sh
$ curl http://localhost:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## Watch log of Pod

```sh
$ kubectl logs nginx
```
```sh
$ kubectl logs -f nginx
```

## Remove PODs

```sh
$ kubectl delete pod wpg-nginx
```

