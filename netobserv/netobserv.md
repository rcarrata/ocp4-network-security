## Network Observability installation

### Without GitOps

* Create namespace for network observability:

```sh
kubectl create ns network-observability
```

* Install Loki (without OLM):

```sh
kubectl apply -k manifests/loki/base
```

* Install Grafana Operator:

```sh
kubectl apply -k manifests/grafana-operator/overlays/operator/base
```

* Deploy Grafana Instance:

```sh
kubectl apply -k manifests/grafana-operator/overlays/instance
```

* Install Network Observability Operator:

```sh
kubectl apply -k manifests/netobserv/operator/base/
```

* Install Network Observability Instance:

```sh
kubectl apply -k netobserv/manifests/netobserv/instance/overlays/default/
```

<img align="center" width="570" src="assets/netobsv.png">

* Network Observability Dashboard (Topology)

<img align="center" width="570" src="assets/network-traffic.png">

* Network Observability Dashboard (Logs)

<img align="center" width="570" src="assets/network-traffic2.png">

* Grafana Dashboard (after importing)

<img align="center" width="570" src="assets/grafana-dashboard.png">

### With GitOps 

* Install OpenShift Pipelines / Tekton:

```bash
until kubectl apply -k bootstrap-argo/; do sleep 2; done
```

* After couple of minutes check the OpenShift GitOps and Pipelines:

```
ARGOCD_ROUTE=$(kubectl get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}{"\n"}')
curl -ks -o /dev/null -w "%{http_code}" https://$ARGOCD_ROUTE
```

* Deploy two Argo App of Apps for deploy Network Observability operators and dependencies automatically:

```sh
kubectl apply -k netobserv/deploy-netobsv-argoapps/
```

* This will deploy two Argo App of Apps that will deploy the operators and the instances of Network Observability, Grafana and Loki:

<img align="center" width="570" src="assets/gitops1.png">

* The first Argo App of Apps will deploy the operators:

<img align="center" width="570" src="assets/gitops2.png">

* The second Argo App of Apps will deploy the instances:

<img align="center" width="570" src="assets/gitops3.png">

* These two Argo App of Apps will deploy and manage all Operators, k8s manifests, and prerequisites needed for deploy the Network Observability Operator:

<img align="center" width="570" src="assets/gitops4.png">

* For example the Grafana Operator Argo App will deploy, manage and sync all the bits and reqs needed for deploy the Grafana instance in the Network Observability namespace:

<img align="center" width="570" src="assets/gitops5.png">

* After a bit, ArgoCD will deploy everything that is needed for the Network Observability demo:

<img align="center" width="570" src="assets/gitops6.png">

## TODO 

* Fix the GrafanaDashboard import using the Grafana Operator
* Use Loki Operator deployment instead of Loki k8s manifests