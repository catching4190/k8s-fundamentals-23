All recordings:

https://capgemini-my.sharepoint.com/personal/oleksandr_farion2_capgemini_com/_layouts/15/onedrive.aspx?id=%2Fpersonal%2Foleksandr%5Ffarion2%5Fcapgemini%5Fcom%2FDocuments%2FRecordings%201&FolderCTID=0x012000F4A2EBF91B875F4488DF85EB0E44BD1C&view=0

# Workshop 1

## Config

```bash
cat ~/.kube/config

kubectl config view
```

## Start minikube

```bash
minikube start
```

### Issue with access to k8s registry

> SOLVED: Docker Desktop should be running, or switch to use QEMU driver

![Issue with access to ](./01/issue-proxy-or-hyperkit.png)

❗  This VM is having trouble accessing https://registry.k8s.io

https://minikube.sigs.k8s.io/docs/handbook/vpn_and_proxy/

```bash
export HTTP_PROXY=http://<proxy hostname:port>
export HTTPS_PROXY=https://<proxy hostname:port>
export HTTPS_PROXY=localhost
export NO_PROXY=localhost,127.0.0.1,10.96.0.0/12,192.168.59.0/24,192.168.49.0/24,192.168.39.0/24
```

Some workaround here, but not help yet:
https://github.com/kubernetes/minikube/issues/7231

```bash
# --image-repository=auto
# --binary-mirror /k8s
# --image-repository=auto
minikube start --alsologtostderr -v=7
```

Maybe try to use other driver

https://minikube.sigs.k8s.io/docs/drivers/

```bash
minikube delete --all --purge
```

### Using Hyperkit

https://minikube.sigs.k8s.io/docs/drivers/hyperkit/

If Docker for Desktop is installed, you already have HyperKit

```bash
minikube start --driver=hyperkit
```

### Using VMWare driver

https://minikube.sigs.k8s.io/docs/drivers/vmware/

https://github.com/machine-drivers/docker-machine-driver-vmware

```bash
brew install docker-machine-driver-vmware
docker-machine create --driver=vmware default # not started, maybe need to install VMWare Fusion
docker-machine rm default

minikube start --driver=vmware

# In case to make it default
minikube config set driver vmware
```

Also this article can be helpful

https://medium.com/rahasak/replace-docker-desktop-with-minikube-and-hyperkit-on-macos-783ce4fb39e3

### Using QEMU

https://minikube.sigs.k8s.io/docs/drivers/qemu/

https://www.qemu.org/download/#macos

```bash
brew install qemu
minikube start --driver=qemu
minikube start --driver qemu --network builtin
minikube start --driver qemu --network builtin --alsologtostderr -v=4
```

### Stop minikube

```bash
minikube stop
minikube delete
```

## Metrics server & dashboard

### On Minikube

#### Metrics server

Won't work, see workarounds

- https://devpress.csdn.net/k8s/62fcbbcbc677032930801d04.html
- https://gist.github.com/F21/08bfc2e3592bed1e931ec40b8d2ab6f5

```bash
minikube addons list

minikube addons enable metrics-server

# --alsologtostderr -v=1 | --url=true
minikube dashboard --url=true


## http://192.168.2.22:8001/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/:

minikube addons disable metrics-server
```

#### Dashboard

https://minikube.sigs.k8s.io/docs/handbook/dashboard/

```bash
minikube dashboard
```

Not working... maybe proxy

### On Docker

#### Metric server

Metrics Server collects resource metrics from kubelets and exposes them in Kubernetes apiserver through Metrics API

https://github.com/kubernetes-sigs/metrics-server

Installation:

https://github.com/kubernetes-sigs/metrics-server#installation

#### Dashboard

General Information:

https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/

Deploying the Dashboard UI:

https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/#deploying-the-dashboard-ui

Accessing the Dashboard UI:

https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/#accessing-the-dashboard-ui

https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md

#### Prepare

You should have [k8s-dashboard-sa.yaml](./01/k8s-dashboard-sa.yaml) file

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# Fix issue with pre-requisites(kubelet CA requirements) 
kubectl patch deployment metrics-server --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/5", "value": "--kubelet-insecure-tls" }]' -n kube-system

kubectl apply –f k8s-dashboard-sa.yaml
```

#### Get token, run proxy and go to dashboard

```bash
kubectl -n kubernetes-dashboard create token admin-user

# After run go to http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
kubectl proxy
```

> Select All namespaces in dropdown to see all of them

# Workshop 2

## General

https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands

- **Create** - Create a resource from a file or from stdin.
- **Apply** - Apply a configuration to a resource by filename or stdin
- **Get** - Display one or many resources
- **Expose** - Take a replication controller, service, deployment or pod and expose it as a new Kubernetes Service
- **Delete** - Delete resources by filenames, stdin, resources and names, or by resources and label selector
- **Label** - Update the labels on a resource
- **Scale** - Set a new size for a Deployment, ReplicaSet or Replication Controller

## Context

```bash
# Context
kubectl config get-contexts
kubectl config current-context
kubectl config use minicube # or docker-desktop
```

## Info

```bash
kubectl get node
kubectl get pods --all-namespaces
kubectl get deployments
kubectl get services
kubectl get events

# Log pods
kubectl logs <pod-name>
```

## Object specs

https://kubernetes.io/docs/concepts/workloads/

```bash
# List of resources
kubectl api-resources --namespaced=true|false

# Api versions
kubectl api-versions

# Documentation
kubectl explain pod
```

## Create / Delete pods

```bash
kubectl create -f kfb-2/dummy-pod.yaml

kubectl get pods

# Delete pod by name
kubectl delete pod dummy-pod

# It can be deleted by filename too
kubectl delete -f kfb-2/dummy-pod.yaml
```

## Deployments

```bash
kubectl apply -f kfb-2/echo-deployment.yaml
kubectl get deploy
kubectl get pods
```

## Exposing

Expose command will create ClusterIP typed service

Requests can be sent to K8S API according to following URL:

http://127.0.0.1:8001/api/v1/namespaces/<NS name>/services/<SVC name>/proxy/

```bash
kubectl expose deployment echo-deployment --port=80 --name=echo-deploy-service

kubectl get services # or svc

kubectl proxy

curl http://127.0.0.1:8001/api/v1/namespaces/default/services/echo-deploy-service/proxy/
```

## Homework

### Task 1

[task1.yaml](./02/task1.yaml)

```bash
kubectl apply -f task1.yaml
kubectl get deploy
kubectl expose deployment task-1-deployment --port=80 --name=task-1-service
kubectl get services
kubectl proxy
```

Then, make request to the service:

```bash
curl http://127.0.0.1:8001/api/v1/namespaces/default/services/task-1-service/proxy/
```

Output:

```json
{
    "host": {
        "hostname": "kubernetes.docker.internal",
        "ip": "::ffff:10.1.0.1",
        "ips": []
    },
    "http": {
        "method": "GET",
        "baseUrl": "",
        "originalUrl": "/",
        "protocol": "http"
    },
    "request": {
        "params": {
            "0": "/"
        },
        "query": {},
        "cookies": {},
        "body": {},
        "headers": {
            "host": "kubernetes.docker.internal:6443",
            "user-agent": "curl/7.87.0",
            "accept": "*/*",
            "accept-encoding": "gzip",
            "x-forwarded-for": "127.0.0.1, 127.0.0.1",
            "x-forwarded-uri": "/api/v1/namespaces/default/services/task-1-service/proxy/"
        }
    },
    "environment": {
        "PATH": "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
        "HOSTNAME": "task-1-deployment-74cb67c874-khdfz",
        "password": "va0s9dv^",
        "username": "bingo",
        "USERNAME": "bingo",
        "PASSWORD": "va0s9dv^",
        "KUBERNETES_SERVICE_HOST": "10.96.0.1",
        "KUBERNETES_SERVICE_PORT": "443",
        "KUBERNETES_SERVICE_PORT_HTTPS": "443",
        "KUBERNETES_PORT": "tcp://10.96.0.1:443",
        "KUBERNETES_PORT_443_TCP": "tcp://10.96.0.1:443",
        "KUBERNETES_PORT_443_TCP_PROTO": "tcp",
        "KUBERNETES_PORT_443_TCP_PORT": "443",
        "KUBERNETES_PORT_443_TCP_ADDR": "10.96.0.1",
        "NODE_VERSION": "16.16.0",
        "YARN_VERSION": "1.22.19",
        "HOME": "/root"
    }
}
```

Cleanup:

```bash
kubectl delete -f task1.yaml
kubectl delete service task-1-service
```

### Task 2

https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/

[task2.yaml](./02/task2.yaml)

Magic url to access to other service from container:

    <service name>.default.svc.cluster.local

Possible commands for "busybox" image:

```bash
nslookup task-1-service.default.svc.cluster.local
# or
wget task-1-service.default.svc.cluster.local
# or
curl task-1-service.default.svc.cluster.local
```

> Notice that each element of `command:` is a argument

Helper to setup cron schedule:

https://crontab.guru/

To run job each 3 minutes:

    0/3 * * * *

```bash
kubectl create -f task1.yaml
kubectl create -f task2.yaml

# Wait 5-6 min
kubectl get jobs --all-namespaces
kubectl get cronjobs
```

Cleanup

```bash
kubectl delete -f task2.yaml
```

# Workshop 3

```bash
kubectl apply -f dep-with-pv.yaml

kubectl get deploy
kubectl get pods

kubectl expose deployment dep-with-pv --port=80 --name=dep-with-pv-service

kubectl get services

kubectl describe service dep-with-pv

kubectl get endpoints

curl http://127.0.0.1:8001/api/v1/namespaces/default/services/dep-with-pv-service/proxy/

# Cleanup

kubectl delete -f dep-with-pv.yaml

kubectl delete service dep-with-pv-service
```

## Task 1

https://kubernetes.io/docs/concepts/configuration/configmap/

```bash
# Deploy
kubectl apply -f v1.yaml
kubectl expose deployment html-deployment --port=80 --name=html-service
kubectl get services
kubectl proxy
curl http://127.0.0.1:8001/api/v1/namespaces/default/services/html-service/proxy/

# Cleanup
kubectl delete service html-service
kubectl delete -f v1.yaml
```

## Task 2

https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistent-volumes

https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims

https://kubernetes.io/docs/reference/labels-annotations-taints/#change-cause

https://julien-chen.medium.com/k8s-how-to-mount-local-directory-persistent-volume-to-kubernetes-pods-of-docker-desktop-for-mac-b72f3ca6b0dd

https://itnext.io/goodbye-docker-desktop-hello-minikube-3649f2a1c469

https://thospfuller.com/2020/12/09/learn-how-to-mount-a-local-drive-in-a-pod-in-minikube-2021/

https://minikube.sigs.k8s.io/docs/handbook/mount/#driver-mounts

https://minikube.sigs.k8s.io/docs/handbook/filesync/

```yaml
hostPath:
    path: "/run/desktop/mnt/host/c/www" #for windows prefix is /run/desktop/mnt/host/<disk-letter>/<local-path-to-folder> 
    path: "/mnt/data" #For linux should be trivial
    path: "/tmp" # For Mac -Filesharing https://julien-chen.medium.com/k8s-how-to-mount-local-directory-persistent-volume-to-kubernetes-pods-of-docker-desktop-for-mac-b72f3ca6b0dd
```

```bash
# ❌  Exiting due to MK_UNIMPLEMENTED: minikube mount is not currently implemented with the builtin network on QEMU, try starting minikube with '--network=socket_vmnet'
minikube mount $HOME/Learning/k8s-23/03/v2:/www

# Workaround: I use solution from here https://minikube.sigs.k8s.io/docs/handbook/filesync/
# just copy files to ~/.minikube/files/www folder and it mounted as /www in the minikube fs

# Check available mounted folders (for file sharing solution, just check your expected path starts from root folder, e.g. /www)
minikube ssh
df -hl

# 1. Deploy
kubectl apply -f v2.yaml
kubectl expose deployment html-deployment --port=80 --name=html-service
kubectl get services
kubectl exec html-deployment-6d969f6859-gmc2v -it -- /bin/sh

kubectl describe deployment html-deployment

kubectl proxy
curl http://127.0.0.1:8001/api/v1/namespaces/default/services/html-service/proxy/

# 2. Change index.html and restart minikube
minikube start

# 3. Change app-revision and resources and redeploy via apply
kubectl apply -f v2.yaml

# Check deployment changes, e.g. labels, requested resources
kubectl describe deployment html-deployment

# Check response
curl http://127.0.0.1:8001/api/v1/namespaces/default/services/html-service/proxy/

# Check rollout history
kubectl rollout history deployment html-deployment

# Rollback
kubectl rollout undo deployment html-deployment

# Check replica sets
kubectl get rs


# Edit in vim
kubectl edit --record deployment html-deployment

# Change annotation
kubectl annotate deployments html-deployment description="My favorite deployment with my app"
kubectl annotate deployments html-deployment kubernetes.io/change-cause="Increase cpu and memo"


# Cleanup
kubectl delete service html-service
kubectl delete -f v2.yaml
```

# Workshop 4 - Helm Charts

https://helm.sh/docs/chart_template_guide/getting_started/


```bash
brew install helm

helm create HtmlApp
```

## Syntax

{{ <operator> <arguments?> <scope?> }}

### Operators

https://helm.sh/docs/chart_template_guide/function_list/

### Inlcude

https://helm.sh/docs/howto/charts_tips_and_tricks/#using-the-include-function

### Variables

https://helm.sh/docs/chart_template_guide/variables/

It gets value from _helpers.tpl > search "HtmlApp.fullname"

```go
name: {{ include "HtmlApp.fullname" . }}
```

### Scope

Via dot argument `.` or `with` operator

```go
spec:
    {{- with .Values.imagePullSecrets }}
    imagePullSecrets:
        {{- toYaml . | nindent 8 }}
    {{- end }}
```

### Whitespaces

https://helm.sh/docs/chart_template_guide/control_structures/#controlling-whitespace

Means remove trailing spaces before or after string

```go
// before
{{- toYaml . | nindent 8 }}
// after
{{ toYaml . | nindent 8 -}}
// both
{{- toYaml . | nindent 8 -}}
```

### Piping

https://helm.sh/docs/chart_template_guide/functions_and_pipelines/#pipelines

```go
{{- include "HtmlApp.labels" . | nindent 4 }}
```

### toYaml

https://helm.sh/docs/howto/charts_tips_and_tricks/#using-the-include-function

### Magic shortcut

In VSCode:

Command + P > Helm: Convert to template parameter

### Check configuration

```bash
helm lint ./HtmlApp
```

### Dry run

```bash
helm install --dry-run chart-test ./HtmlApp
```

### Install

```bash
helm install chart-test ./HtmlApp
```

### Uninstall

```bash
helm uninstall chart-test
```

### Upgrade

```bash
# override value
helm upgrade chart-test ./HtmlApp --set "replicaCount=2"

# override by file and, additionally, by value
helm upgrade chart-test ./HtmlApp -f ./HtmlApp/values-dev.yaml --set "replicaCount=2"
```
