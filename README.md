# Securing Cloud-Native Workloads: Hands-On with Notary Project, ORAS, and Ratify.

## Background

Securing your software supply chain for container images

## Why ensuring the integrity and authenticity?

Authenticity:

Integrity:

## The end-to-end solution

<Picture>

## Introduction on CNCF projects:

Notary Project
ORAS
Ratify

## Hands-on

1. Set up environment
2. Build your container image
3. Choose your Key Management System (KMS)
4. Sign a container image
5. Publish your container image and sigantures to production
6. Verify your container image during deployment

### Set up environement

Signing:
- Notary Project tooling Notation CLI

Publishing:
- ORAS

Verification:
- OPA Gatekeeper
- Ratify
- Helm

Docker Desktop

### Build your container image

```shell
export IMAGE=docker.io/yizha1/net-monitor:v1
docker buildx build . -f Dockerfile -o type=oci,dest=net-monitor.tar -t $IMAGE
mkdir net-monitor
tar -xf net-monitor.tar -C net-monitor
```

### Choose your KMS

```shell
notation cert generate-test mycompany.io --default
notation cert ls
notation key ls
```

### List signatures

```shell
export NOTATION_EXPERIMENTAL=1
notation ls --oci-layout net-monitor:v1
```

### Sign the image

```shell
notation sign --oci-layout ./net-monitor:v1
```

### List signatures

```shell
notation list --oci-layout ./net-monitor:v1
```

## Publishing both the container image and signature


```shell
oras cp -r ./net-monitor:v1 --from-oci-layout docker.io/yizha1/net-monitor:v1
```

### View from docker hub

Switch to [my docker hub](https://hub.docker.com/repositories/yizha1)


## Verify a container image in CI/CD pipelines.

Note: Use Notation CLI to simulate CI/CD pipelines

```json
{
 "version": "1.0",
 "trustPolicies": [
    {
         "name": "mypolicy",
         "registryScopes": [ "docker.io/yizha1/net-monitor" ],
         "signatureVerification": {
             "level" : "strict"
         },
         "trustStores": [ "ca:mycert" ],
         "trustedIdentities": [
             "x509.subject: CN=2.23.io,O=Notary,L=Seattle,ST=WA,C=US"
         ]
     }
 ]
}
```

```shell
notation policy import mypolicy.json
notation policy show
notation verify $IMAGE
```

## Verify a container image before pulling to K8s

### Install Gatekeeper

```shell
helm repo add gatekeeper  https://open-policy-agent.github.io/gatekeeper/charts
helm install gatekeeper/gatekeeper --name-template=gatekeeper --namespace gatekeeper-system --create-namespace --set enableExternalData=true --set validatingWebhookTimeoutSeconds=5 --set mutatingWebhookTimeoutSeconds=2 --set externaldataProviderResponseCacheTTL=10s
```

### Install Ratify

Get your root CA certificate:

```shell
cp ~/.config/notation/localkeys/mycompany.io.crt .
```

```shell
helm repo add ratify https://ratify-project.github.io/ratify
helm install ratify ratify/ratify --atomic --namespace gatekeeper-system --set-file notationCerts[0]="mycompany.io.crt" --set featureFlags.RATIFY_CERT_ROTATION=true --set notation.trustPolicies[0].registryScopes[0]="docker.io/yizha1/net-monitor" --set notation.trustPolicies[0].trustStores[0]=ca:notationCerts[0] --set notation.trustPolicies[0].trustedIdentities[0]="x509.subject: CN=mycompany.io\,O=Notary\,L=Seattle\,ST=WA\,C=US"
```

### Set up policies

```shell
# You can customize your policy based using Rego language
kubectl apply -f https://ratify-project.github.io/ratify/library/default/template.yaml

# Deny or Warn
kubectl apply -f https://ratify-project.github.io/ratify/library/default/samples/constraint.yaml
```

## Check the results

### Verify an unsigned image

```shell
export IMAGE_UNSIGNED=docker.io/yizha1/unsigned:v1
kubectl run demo-unsigned --image=$IMAGE_UNSIGNED
```
The result is per your definition of your policy.

Check the log of Ratify

### Verify a signed image

```shell
kubectl run demo-signed --image=$IMAGE_SIGNED
```

## Next Steps



