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

### Using podman

https://podman.io/docs/installation

https://podman-desktop.io/downloads

```bash
brew install podman

# It download the fedora-coreos-38.20230527.2.0-qemu.x86_64.qcow2.xz first time
podman machine init

# or you can set your specs
podman machine init --cpus 2 --memory 12288 --disk-size 25

# Need to do each time before minikube start
podman machine start

# Override VM configuration on existing VM
# https://docs.podman.io/en/latest/markdown/podman-machine-set.1.html
podman machine stop
podman machine set cpus=4
podman machine start

# https://minikube.sigs.k8s.io/docs/drivers/podman/#known-issues
podman system connection list
podman system connection default podman-machine-default-root

# Also volumes will be mounted there:
# Mounting volume... /Users:/Users
# Mounting volume... /private:/private
# Mounting volume... /var/folders:/var/folders

podman info

# Remove
podman machine stop
podman machine rm
brew remove podman
```

https://minikube.sigs.k8s.io/docs/drivers/podman/

```bash
minikube start --driver=podman

# It’s recommended to run minikube with the podman driver and CRI-O container runtime (except when using Rootless Podman)
minikube start --driver=podman --memory 8192 --cpus=4 --container-runtime=cri-o

# To make podman the default driver:
minikube config set driver podman

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

# Workshop 5 - Debug

```bash
# Try to simulate install
helm install kfb5 ./kfb5 --dry-run # or use --generate-name instead kfb5

# Render and output template even if error
helm template --dry-run --debug kfb5 ./kfb5 > output.yaml

# Fix indent in yaml file and install
helm install kfb5 ./kfb5

# Error: INSTALLATION FAILED: 1 error occurred:
#     * Deployment.apps "kfb5-kfb5" is invalid: spec.template.spec.containers[0].resources.requests: Invalid value: "1128Mi": must be less than or equal to memory limit

# Fix memory value in values.yaml and try again
helm install kfb5 ./kfb5

# Error: INSTALLATION FAILED: cannot re-use a name that is still in use

# Will use upgrade
helm upgrade kfb5 ./kfb5

# Release "kfb5" has been upgraded. Happy Helming!
# NAME: kfb5
# LAST DEPLOYED: Thu May 25 13:33:11 2023
# NAMESPACE: default
# STATUS: deployed
# REVISION: 2
# TEST SUITE: None
# NOTES:
# 1. Get the application URL by using command 
# kubectl port-forward service/kfb5 80:80


# Check physical recourses we have
kubectl describe node
kubectl top node

# Fix cpu values 64 -> 1
helm upgrade kfb5 ./kfb5 --install

# Release "kfb5" has been upgraded. Happy Helming!
# NAME: kfb5
# LAST DEPLOYED: Thu May 25 13:46:05 2023
# NAMESPACE: default
# STATUS: deployed
# REVISION: 3
# TEST SUITE: None
# NOTES:
# 1. Get the application URL by using command 
# kubectl port-forward service/kfb5 80:80

# Check
kubectl get deploy

# NAME        READY   UP-TO-DATE   AVAILABLE   AGE
# kfb5-kfb5   0/1     1            0           14m

# Why here is 2 pods?
kubectl get pods

# NAME                         READY   STATUS                            RESTARTS   AGE
# kfb5-kfb5-556bd849c-lnz52    0/1     Pending                           0          13m
# kfb5-kfb5-6cdd9f564f-p4c8q   0/1     Init:CreateContainerConfigError   0          29s

kubectl describe pod kfb5-kfb5-6cdd9f564f-p4c8q 

# Name:             kfb5-kfb5-6cdd9f564f-p4c8q
# Namespace:        default
# Priority:         0
# Service Account:  default
# Node:             minikube/10.0.2.15
# Start Time:       Thu, 25 May 2023 13:46:06 +0300
# Labels:           app.kubernetes.io/instance=kfb5
#                   app.kubernetes.io/name=kfb5
#                   pod-template-hash=6cdd9f564f
# Annotations:      <none>
# Status:           Pending
# IP:               10.244.0.236
# IPs:
#   IP:           10.244.0.236
# Controlled By:  ReplicaSet/kfb5-kfb5-6cdd9f564f
# Init Containers:
#   init:
#     Container ID:  
#     Image:         busybox
#     Image ID:      
#     Port:          <none>
#     Host Port:     <none>
#     Command:
#       /bin/sh
#       -c
#     Args:
#       if [[ -z "${LOG_LEVEL}" ]]; then echo "Failed to init. LOG_LEVEL env var should not be empty."; exit 1; else echo "Init successful"; exit 0; fi
#     State:          Waiting
#       Reason:       CreateContainerConfigError
#     Ready:          False
#     Restart Count:  0
#     Environment:
#       LOG_LEVEL:  <set to the key 'log_level' of config map 'kfb5-kfb5-configmap'>  Optional: false
#     Mounts:
#       /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-d6pks (ro)
# Containers:
#   kfb5:
#     Container ID:   
#     Image:          ngnix:1.16.0
#     Image ID:       
#     Port:           8080/TCP
#     Host Port:      0/TCP
#     State:          Waiting
#       Reason:       PodInitializing
#     Ready:          False
#     Restart Count:  0
#     Limits:
#       cpu:     1
#       memory:  128Mi
#     Requests:
#       cpu:        1
#       memory:     128Mi
#     Environment:  <none>
#     Mounts:
#       /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-d6pks (ro)
# Conditions:
#   Type              Status
#   Initialized       False 
#   Ready             False 
#   ContainersReady   False 
#   PodScheduled      True 
# Volumes:
#   kube-api-access-d6pks:
#     Type:                    Projected (a volume that contains injected data from multiple sources)
#     TokenExpirationSeconds:  3607
#     ConfigMapName:           kube-root-ca.crt
#     ConfigMapOptional:       <nil>
#     DownwardAPI:             true
# QoS Class:                   Burstable
# Node-Selectors:              <none>
# Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
#                              node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
# Events:
#   Type     Reason     Age                    From               Message
#   ----     ------     ----                   ----               -------
#   Normal   Scheduled  5m3s                   default-scheduler  Successfully assigned default/kfb5-kfb5-6cdd9f564f-p4c8q to minikube
#   Normal   Pulled     4m54s                  kubelet            Successfully pulled image "busybox" in 7.727850919s (7.727854755s including waiting)
#   Normal   Pulled     4m52s                  kubelet            Successfully pulled image "busybox" in 1.722123761s (1.722134242s including waiting)
#   Normal   Pulled     4m37s                  kubelet            Successfully pulled image "busybox" in 1.707186746s (1.707255842s including waiting)
#   Normal   Pulled     4m22s                  kubelet            Successfully pulled image "busybox" in 1.698566817s (1.698571241s including waiting)
#   Normal   Pulled     4m7s                   kubelet            Successfully pulled image "busybox" in 1.663885208s (1.663891481s including waiting)
#   Normal   Pulled     3m55s                  kubelet            Successfully pulled image "busybox" in 1.751082154s (1.751151929s including waiting)
#   Normal   Pulled     3m41s                  kubelet            Successfully pulled image "busybox" in 1.724097364s (1.724169661s including waiting)
>>> #   Warning  Failed     3m25s (x8 over 4m54s)  kubelet            Error: couldn't find key log_level in ConfigMap default/kfb5-kfb5-configmap
#   Normal   Pulled     3m25s                  kubelet            Successfully pulled image "busybox" in 1.690343833s (1.690416543s including waiting)
#   Normal   Pulling    3m14s (x9 over 5m2s)   kubelet            Pulling image "busybox"

# Fix Error: couldn't find key log_level in ConfigMap in values.yaml
# Read the package docs from registry, but in our case we can check deployment.yaml:37
# IMPORTANT: take log_level value in quote ""
helm upgrade kfb5 ./kfb5 --install

# Release "kfb5" has been upgraded. Happy Helming!
# NAME: kfb5
# LAST DEPLOYED: Thu May 25 13:57:03 2023
# NAMESPACE: default
# STATUS: deployed
# REVISION: 4
# TEST SUITE: None
# NOTES:
# 1. Get the application URL by using command 
# kubectl port-forward service/kfb5 80:80

# Check again
kubectl get pods
kubectl describe pod

# And ... fix image.repository
helm upgrade kfb5 ./kfb5 --install

# Check one more time
kubectl get pods

# NAME                         READY   STATUS    RESTARTS   AGE
# kfb5-kfb5-6fd5f87f9b-gsgrg   0/1     Pending   0          9s
# kfb5-kfb5-7766598c44-d7b6x   1/1     Running   0          2m50s

# Sometimes, "0/1 nodes are available: 1 Insufficient cpu." happens for pending nodes. Try to iteratively reduce requests/limits cpu and re-install chart, or temporary scale up node.
kubectl derscibe pod kfb5-kfb5-6fd5f87f9b-gsgrg

# Stuck with forwarding port
kubectl port-forward service/kfb5 80:80

# Error from server (NotFound): services "kfb5" not found

# Lets check services
kubectl get services

# NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# kfb5-kfb5    ClusterIP   10.108.219.51   <none>        80/UDP    52m
# kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   13d

kubectl port-forward service/kfb5-kfb5 80:80

# error: UDP protocol is not supported for 80

# Set services.protocol = TCP in values.yaml and deploy again
helm upgrade kfb5 ./kfb5 --install

# Error port not found, why 8080?
kubectl port-forward service/kfb5-kfb5 80:80

# Unable to listen on port 80: Listeners failed to create with the following errors: [unable to create listener: Error listen tcp4 127.0.0.1:80: bind: permission denied unable to create listener: Error listen tcp6 [::1]:80: bind: permission denied]
# error: unable to listen on any of the requested ports: [{80 8080}]

# Lets fix deployment.containerPort = 80 and deploy again
helm upgrade kfb5 ./kfb5 --install

# Check port in containers
kubectl describe pod 


# Wrong page (default nginx) displayed. Check values.yaml -> storage.enabled, claimName in configmap.yaml is wrong (should be just name), and fix deployment.volumeName _ -> -

helm list

Release "kfb5" has been upgraded. Happy Helming!
NAME: kfb5
LAST DEPLOYED: Thu May 25 17:04:22 2023
NAMESPACE: default
STATUS: deployed
REVISION: 14
TEST SUITE: None
NOTES:
1. Get the application URL by using command 
kubectl port-forward service/kfb5 80:80

```

# Workshop 6

Prerequirements:

1. Setup and use Podman as driver for minikube
2. Ensure cluster use increased memory limints and cpus=4. Recreate it if needed.

```bash
minikube start --driver=podman --memory 8192
```

## OpenSearch

Opensearch – open source alternative ElasticSearch

```bash
# Add repo
helm repo add opensearch https://opensearch-project.github.io/helm-charts/

# Do it always
helm repo update

# Installing opensearch backend
helm install opensearch opensearch/opensearch -n logging --create-namespace

# -w means watch
kubectl get pods --namespace=logging -l app.kubernetes.io/component=opensearch-cluster-master -w

# Troubleshooting
kubectl describe pods opensearch-cluster-master-0 --namespace=logging
kubectl logs opensearch-cluster-master-0 --namespace=logging

# ERROR: [1] bootstrap checks failed
# [1]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
# ERROR: OpenSearch did not exit normally - check the logs at /usr/share/opensearch/logs/opensearch-cluster.log

# See requirements: https://opensearch.org/downloads.html

# Solution: https://github.com/kubernetes/minikube/issues/1306
minikube ssh 'echo "sysctl -w vm.max_map_count=262144" | sudo tee -a /var/lib/boot2docker/bootlocal.sh'

# Also try to run in single node mode with 1 replicas. To do so:

# Output values to file
helm show values opensearch/opensearch > values.yaml

# Edit file, and set singleNode: true and replicas: 1, maybe edit requested resources
# Upgrade, using values.yaml
helm upgrade opensearch opensearch/opensearch -f ./values.yaml -n logging

# Info
helm show values opensearch/opensearch # > values.yaml

kubectl get persistentvolumes

kubectl get services -n logging
# NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
# opensearch-cluster-master            ClusterIP   10.105.187.56   <none>        9200/TCP,9300/TCP            60m
# opensearch-cluster-master-headless   ClusterIP   None            <none>        9200/TCP,9300/TCP,9600/TCP   60m

# Check connection
kubectl exec --stdin --tty -n logging opensearch-cluster-master-0 -- /bin/bash
curl -XGET https://localhost:9200 -u 'admin:admin' --insecure
# curl: (7) Failed to connect to localhost port 9200 after 1216 ms: Couldn't connect to server

# Forward port from services and check again
kubectl -n logging port-forward service/opensearch-cluster-master 8080:9200

curl -XGET https://localhost:8080 -u 'admin:admin' --insecure
# OpenSearch Security not initialized.%   

# Installing opensearch UI
helm install opensearch-dashboards opensearch/opensearch-dashboards -n logging

export POD_NAME=$(kubectl get pods --namespace logging -l "app.kubernetes.io/name=opensearch-dashboards,app.kubernetes.io/instance=opensearch-dashboards" -o jsonpath="{.items[0].metadata.name}")
export CONTAINER_PORT=$(kubectl get pod --namespace logging $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
echo "Visit http://127.0.0.1:8080 to use your application"
kubectl --namespace logging port-forward $POD_NAME 8200:$CONTAINER_PORT

# Forward port to dashboard after pod starts
kubectl get services -n logging
# NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
# opensearch-cluster-master            ClusterIP   10.105.187.56   <none>        9200/TCP,9300/TCP            76m
# opensearch-cluster-master-headless   ClusterIP   None            <none>        9200/TCP,9300/TCP,9600/TCP   76m
# opensearch-dashboards                ClusterIP   10.101.148.63   <none>        5601/TCP                     39m


kubectl -n logging port-forward service/opensearch-dashboards 8081:5601
```

## FluentBit

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update

helm upgrade fluent-bit fluent/fluent-bit -f fluentbit_values.yaml -n logging --install

# Open localhost:8200 use admin:admin and go to Dashboard and create index pattern,
# then observe your logs and enjoy
```

## Telepresence

Set up your ideal development environment for Kubernetes in seconds. Accelerate your inner development loop with hot reload using your existing IDE, and workflow.

https://www.getambassador.io/docs/telepresence/latest/install

```bash
brew install datawire/blackbird/telepresence

telepresence helm install

telepresence connect

telepresence dashboard # login ?

telepresence status

# Run app from 05/kfb5
helm install kfb5 ./kfb5

telepresence list

telepresence intercept kfb5 --port 8123:http --env-file example-service-intercept.env

# Then run some server on localhost, that listen port 8123 and check output on localhost:8080
node app.js

# Disable intercepting anc check output on localhost:8080 again
telepresence leave
```
