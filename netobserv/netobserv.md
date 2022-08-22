## Network Observability installation

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

## TODO 

* Fix the GrafanaDashboard import using the Grafana Operator
* Use Loki Operator deployment instead of Loki k8s manifests