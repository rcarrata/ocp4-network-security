# Sign Images with Sigstore

## Bootstrap

* Install OpenShift Pipelines / Tekton:

```bash
kubectl apply -f bootstrap/openshift-pipelines-operator-subscription.yaml
```

## Testing Cosign

```
podman pull quay.io/centos7/httpd-24-centos7:20220713
podman tag quay.io/centos7/httpd-24-centos7:20220713 ghcr.io/${USERNAME}/centos7/httpd-24-centos7:0.1
podman push  ghcr.io/${USERNAME}/httpd-24-centos7:0.1

cosign sign --key cosign.key ghcr.io/${USERNAME}/httpd-24-centos7:0.1

cosign verify --key cosign.pub ghcr.io/${USERNAME}/httpd-24-centos7:0.1
```

## Quay.io Repository Setup

* Add a Public Quay Repository
* Add Robot Account and assign Write or Admin permissions to this Quay Repository
* Grab the QUAY_TOKEN and the USERNAME that is provided

* http://docs.quay.io/issues/no-create-permission.html
* https://jaland.github.io/tekton/2021/01/26/tekton-openshift.html
* http://docs.quay.io/guides/repo-permissions.html

## Deploy Tekton Pipeline and Tasks

* Export the token for the GitHub Registry / ghcr.io:

```bash
export QUAY_TOKEN=""
export EMAIL="xxx"
export USERNAME="rcarrata+acs_integration"
export NAMESPACE="demo-sign"
```

* Generate a docker-registry secret with the credentials for GitHub Registry to push/pull the images and signatures:

```bash
kubectl create secret docker-registry regcred --docker-server=quay.io --docker-username=${USERNAME} --docker-email=${EMAIL}--docker-password=${QUAY_TOKEN} -n ${NAMESPACE}
```

* Add the imagePullSecret to the ServiceAccount “pipeline” in the namespace of the demo:

```
export SERVICE_ACCOUNT_NAME=pipeline
kubectl patch serviceaccount $SERVICE_ACCOUNT_NAME \
 -p "{\"imagePullSecrets\": [{\"name\": \"regcred\"}]}" -n $NAMESPACE
```

* Deploy all the Tekton Tasks and Pipelines for run this demo:

```bash
kubectl apply -k pipelines/
```

## Generate Sigstore KeyPairs

* Generate a pki key-pair for signing with cosign:

```bash
export COSIGN_PASSWORD=redhat
cosign generate-key-pair k8s://${NAMESPACE}/cosign
```

## Generate API Token within Stackrox

* https://docs.openshift.com/acs/3.70/integration/integrate-with-ci-systems.html#cli-authentication_integrate-with-ci-systems

```
export ROX_API_TOKEN="xxx"
roxctl --insecure-skip-tls-verify image check --endpoint central-stackrox.apps.ocp4.rcarrata.com:443 --image quay.io/centos7/httpd-24-centos7:centos7
```

## Add Signature Integration within ACS

* https://docs.openshift.com/acs/3.70/integration/integrate-with-ci-systems.html#cli-authentication_integrate-with-ci-systems

## Run Signed Pipeline

* Run the pipeline for build the image, push to the GitHub registry, sign the image with cosign, push the signature of the image to the GitHub registry:

```bash
kubectl create -f run/sign-images-pipelinerun.yaml
```

* This pipeline will deploy the signed image and also will be validated against kyverno cluster policy:

```bash
k get deploy -n workshop pipelines-vote-api
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
pipelines-vote-api   1/1     1            1           29h
```

<img align="center" width="570" src="assets/signed-1.png">

## Run Unsigned Pipeline

* Run the pipeline for build the image and push to the GitHub registry, but this time without sign with cosign private key:

```bash
kubectl create -f run/unsigned-images-pipelinerun.yaml
```

* The pipeline will fail because, in the last step the pipeline will try to deploy the image, and the Kyverno Cluster Policy will deny the request because it's not signed with the cosign step using the same private key as the pipeline before:

<img align="center" width="970" src="assets/unsigned-1.png">

* As we can see the error that the pipeline outputs it's due to the Kyverno Cluster Policy with the image signature mismatch error:

<img align="center" width="570" src="assets/unsigned-2.png">

## TSHOOT

* [Using Argo to deploy, baseline policies are constantly out-of-sync](https://github.com/kyverno/kyverno/issues/2234)
