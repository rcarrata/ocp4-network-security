# Sign Images with Sigstore

## Bootstrap

* Install OpenShift Pipelines / Tekton:

```bash
kubectl apply -f bootstrap/openshift-pipelines-operator-subscription.yaml
```

* Install Kyverno though Helm:

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno --namespace kyverno kyverno/kyverno --create-namespace
```

* Check that Kyverno pods are Running state:

```bash
kubectl get pod -n kyverno
NAME                      READY   STATUS    RESTARTS   AGE
kyverno-55f86d8cd-69pzv   1/1     Running   0          2m3s
```

## Deploy Tekton Pipeline and Tasks

* Export the token for the GitHub Registry / ghcr.io:

```bash
export PAT_TOKEN="xxx"
export EMAIL="xxx"
export USERNAME="rcarrata"
export NAMESPACE="workshop"
```

* Generate a docker-registry secret with the credentials for GitHub Registry to push/pull the images and signatures:

```bash
kubectl create secret docker-registry ghcr-auth-secret --docker-server=ghcr.io --docker-username=${USERNAME} --docker-email=${EMAIL}--docker-password=${PAT_TOKEN} -n ${NAMESPACE}
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

* Generate a docker-registry secret with the credentials for GitHub Registry to allow the Kyverno to download the signature from GitHub registry in order to verify the image:

```bash
kubectl create secret docker-registry regcred --docker-server=ghcr.io --docker-username=${USERNAME} --docker-email=${EMAIL}--docker-password=${PAT_TOKEN} -n kyverno
```

* Update the kyverno imagepullsecrets to include the registry creds:

```bash
kubectl get deploy kyverno -n kyverno -o yaml | grep containers -A5
      containers:
      - args:
        - --imagePullSecrets=regcred
        env:
        - name: INIT_CONFIG
          value: kyverno
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

<img align="center" width="470" src="assets/signed-1.png">

## Run Unsigned Pipeline

* Run the pipeline for build the image and push to the GitHub registry, but this time without sign with cosign private key:

```bash
kubectl create -f run/unsigned-images-pipelinerun.yaml
```

* The pipeline will fail because, in the last step the pipeline will try to deploy the image, and the Kyverno Cluster Policy will deny the request because it's not signed with the cosign step using the same private key as the pipeline before:

<img align="center" width="470" src="assets/unsigned-1.png">

* As we can see the error that the pipeline outputs it's due to the Kyverno Cluster Policy with the image signature mismatch error:

<img align="center" width="470" src="assets/unsigned-2.png">
