# openfaas-flux

OpenFaaS Cluster state managed by Weave Flux

### Install Weave Flux 

Add Weave Flux chart repo:

```bash
helm repo add sp https://stefanprodan.github.io/k8s-podinfo
```

Install Weave Flux and its Helm Operator by specifying your fork URL 
(replace `stefanprodan` with your GitHub username): 

```bash
helm install --name cd \
--set helmOperator.create=true \
--set git.url=git@github.com:stefanprodan/openfaas-flux \
--set git.chartsPath=charts \
--namespace flux \
sp/weave-flux
```

You can connect Weave Flux to Weave Cloud using your service token:

```bash
helm install --name cd \
--set token=YOUR_WEAVE_CLOUD_SERVICE_TOKEN \
--set helmOperator.create=true \
--set git.url=git@github.com:stefanprodan/openfaas-flux \
--set git.chartsPath=charts \
--namespace flux \
sp/weave-flux
```

### Setup Git sync

At startup Flux generates a SSH key and logs the public key. 
Find the SSH public key with:

```bash
export FLUX_POD=$(kubectl get pods --namespace flux -l "app=weave-flux,release=cd" -o jsonpath="{.items[0].metadata.name}")
kubectl -n flux logs $FLUX_POD | grep identity.pub | cut -d '"' -f2 | sed 's/.\{2\}$//'
```

In order to sync your cluster state with git you need to copy the public key and 
create a **deploy key** with **write access** on your GitHub repository.

Open GitHub, navigate to your fork, go to _Setting > Deploy keys_ click on _Add deploy key_, check 
_Allow write access_, paste the Flux public key and click _Add key_.

After a couple of seconds Flux
* creates the `openfaas` and `openfaas-fn` namespaces
* installs OpenFaaS Helm release
* creates the OpenFaaS functions

Check OpenFaaS services deployment status:

```
kubectl -n openfaas get deployments
NAME           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
alertmanager   1         1         1            1           1m
gateway        1         1         1            1           1m
nats           1         1         1            1           1m
prometheus     1         1         1            1           1m
queue-worker   1         1         1            1           1m
```

You can access the OpenFaaS Gateway using the NodePort service:

```
kubectl -n openfaas get svc gateway-external
NAME               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
gateway-external   NodePort   10.27.247.217   <none>        8080:31112/TCP   10h
```

Check OpenFaaS functions deployment status:

```
kubectl -n openfaas-fn get pods
NAME                        READY     STATUS    RESTARTS   AGE
nodeinfo-58c5bd8998-hxd4g   1/1       Running   0          26s
nodeinfo-58c5bd8998-v5zw6   1/1       Running   0          26s
```

### Manage Helm releases with Weave Flux

The Flux Helm operator provides an extension to Weave Flux to be able to automate Helm Chart releases.
A Chart release is described through a Kubernetes custom resource named `FluxHelmRelease`.
The Flux daemon synchronises these resources from git to the cluster,
and the Flux Helm operator makes sure Helm charts are released as specified in the resources.

![helm](docs/screens/flux-helm.png)

OpenFaaS release definition:

```yaml
apiVersion: helm.integrations.flux.weave.works/v1alpha2
kind: FluxHelmRelease
metadata:
  name: openfaas
  namespace: openfaas
  labels:
    chart: openfaas
spec:
  chartGitPath: openfaas
  releaseName: openfaas
  values:
    serviceType: NodePort
    rbac: true
    images:
      gateway: functions/gateway:0.8.0
      prometheus: prom/prometheus:v2.2.0
      alertmanager: prom/alertmanager:v0.15.0-rc.1
      nats: nats-streaming:0.6.0
      queueWorker: functions/queue-worker:0.4.3
      operator: functions/faas-o6s:0.4.0
```

You can use kubectl to list Flux Helm releases:

```
kubectl -n openfaas get FluxHelmReleases
NAME       AGE
openfaas   1m
```

### Manage OpenFaaS functions with Weave Flux

An OpenFaaS function is describe through a Kubernetes custom resource named `function`.
The Flux daemon synchronises these resources from git to the cluster,
and the OpenFaaS Operator creates the function Kubernetes deployments and ClusterIP services as specified in the resources.

![functions](docs/screens/flux-openfaas.png)

OpenFaaS function definition:

```yaml
apiVersion: o6s.io/v1alpha1
kind: Function
metadata:
  name: certinfo
  namespace: openfaas-fn
spec:
  name: certinfo
  replicas: 1
  image: stefanprodan/certinfo
  limits:
    cpu: "100m"
    memory: "128Mi"
  requests:
    cpu: "10m"
    memory: "64Mi"
```

You can use kubectl to list OpenFaaS functions:

```
kubectl -n openfaas-fn get functions
NAME       AGE
certinfo   1m
nodeinfo   1m
```
