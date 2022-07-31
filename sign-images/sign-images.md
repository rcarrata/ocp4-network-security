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

cosign sign --key cosign.key quay.io/${USERNAME}/httpd-24-centos7:0.1

cosign verify --key cosign.pub quay.io/${USERNAME}/httpd-24-centos7:0.1
```

## Quay.io Repository Setup

* Add a Public Quay Repository
* Add Robot Account and assign Write or Admin permissions to this Quay Repository
* Grab the QUAY_TOKEN and the USERNAME that is provided

* http://docs.quay.io/issues/no-create-permission.html
* https://jaland.github.io/tekton/2021/01/26/tekton-openshift.html
* http://docs.quay.io/guides/repo-permissions.html

## Deploy Tekton Pipeline and Tasks

* Export the token for the GitHub Registry / quay.io:

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

oc secrets link pipeline regcred -n demo-sign
oc secrets link default regcred -n demo-sign
```

* Deploy all the Tekton Tasks and Pipelines for run this demo:

```bash
kubectl apply -k manifests/
```

## Generate Sigstore KeyPairs

* Generate a pki key-pair for signing with cosign:

```bash
export COSIGN_PASSWORD=redhat
cosign generate-key-pair k8s://${NAMESPACE}/cosign

kubectl get secret -n demo-sign cosign -o jsonpath="{.data.cosign\.key}" | base64 -d >> cosign.key
```

## Generate API Token within Stackrox

* https://docs.openshift.com/acs/3.70/integration/integrate-with-ci-systems.html#cli-authentication_integrate-with-ci-systems

- Platform Configuration → Integrations → Authentication Tokens → API Token

```
export ROX_API_TOKEN="xxx"
roxctl --insecure-skip-tls-verify image check --endpoint central-stackrox.apps.ocp4.rcarrata.com:443 --image quay.io/centos7/httpd-24-centos7:centos7
```

## Add Signature Integration within ACS

* https://docs.openshift.com/acs/3.70/operating/verify-image-signatures.html#configure-signature-integration_verify-image-signatures

```
https://central-stackrox.apps.xxx.xxx.com/main/integrations/signatureIntegrations/signature/create
```

- Platform Configuration → Integrations -> Signature Integrations -> Integrate:

* signature_verification
* cosign_pubkey
* xxx

## Add Policy Image Signature Verification

* To Upload with the ACS API:

```
https://{{ acs_url }}/v1/policies/import
```

* Create the ACS Policy with:

https://docs.openshift.com/acs/3.70/operating/manage-security-policies.html#create-policy-from-system-policies-view_manage-security-policies

* Check with the command:

```
roxctl --insecure-skip-tls-verify image check --endpoint central-stackrox.apps.ocp4.rcarrata.com:443 --image quay.io/centos7/httpd-24-centos7:centos7
```

## Integrate and configure Quay.io registry into ACS

* For Quay.io use the Generic Docker Integration:

* https://docs.openshift.com/acs/3.70/integration/integrate-with-image-registries.html#manual-configuration-image-registry-ocp_integrate-with-image-registries

## Create API Token for Tekton Pipeline to integrate in ACS

* https://docs.openshift.com/acs/3.70/cli/getting-started-cli.html#cli-authentication_cli-getting-started

- Platform Configuration -> Integrations -> Authentication Tokens -> API Token

* TODO: Check the output of the rox_api_token and the central endpoint:

```
ROX_API_PIPELINE="xxx"
ROX_HOST=$(kubectl get route -n stackrox central -o jsonpath='{.spec.host}':443)

cat > /tmp/roxsecret.yaml << EOF
apiVersion: v1
data:
  rox_api_token: $(echo $ROX_API_PIPELINE | base64 -w0)
  rox_central_endpoint: $(echo $ROX_HOST)
kind: Secret
metadata:
  name: roxsecrets
  namespace: ${NAMESPACE}
type: Opaque
EOF

kubectl apply -f /tmp/roxsecret.yaml
```

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

<img align="center" width="570" src="assets/signed-2.png">

## Run Unsigned Pipeline

* Run the pipeline for build the image and push to the GitHub registry, but this time without sign with cosign private key:

```bash
kubectl create -f run/unsigned-images-pipelinerun.yaml
```

* TODO

<img align="center" width="570" src="assets/unsigned-1.png">

* TODO

<img align="center" width="570" src="assets/unsigned-2.png">

* TODO

<img align="center" width="570" src="assets/unsigned-2.png">

## TSHOOT

* [Using Argo to deploy, baseline policies are constantly out-of-sync](https://github.com/kyverno/kyverno/issues/2234)



