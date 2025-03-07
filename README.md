# Securing Cloud-Native Workloads: Hands-On with Notary Project, ORAS, and Ratify.

## Background

Software supply chain attacks are rapidly increasing. In cloud-native world, securing your software supply chain for containerized workloads is critical to prevent these escalating threats.

## Why ensuring the integrity and authenticity?

Authenticity: 

How can I ensure images are from trusted identities?

Integrity: 

How can I ensure images are not modified since built?

## The end-to-end solution

The big picture:

KubeConNA 2024 keynote: https://youtu.be/YQuzhSGGPNk

<img width="1056" alt="image" src="https://github.com/user-attachments/assets/9a41b29d-f567-4d3d-9cb1-33cbcfebe279" />


## Hands-on:

<img width="596" alt="image" src="https://github.com/user-attachments/assets/f67fd602-24ad-42fb-ac40-7be1700ee7cb" />

### Introduction on CNCF projects:

- [Notary Project](https://notaryproject.dev/): to provide a cross-industry standardized tools for securing software supply chains by ensuring integrity and authenticity of container images and other artifacts.
- [ORAS](https://oras.land/)
- [Ratify](https://ratify.dev/): At Ratify, our mission is to safeguard the container supply chain by ratifying trustworthy and compliant artifacts. We achieve this through a robust and pluggable verification engine that includes built-in verifiers. 

### Steps

1. Set up environment
2. Build your container image to OCI image layout
4. Choose your Key Management System (KMS)
5. Sign a container image
6. Publish your container image and sigantures to production
7. Verify your container image during deployment

### Get prepared

Docker Desktop & WSL2

Signing:
- Notation CLI, Notary Project tooling

Publishing:
- ORAS CLI

Verification:
- OPA Gatekeeper
- Ratify
- Helm

### Build your container image

Knowledge points:
- [OCI](https://github.com/opencontainers): The Open Container Initiative (OCI) is a Linux Foundation project that aims to establish open standards for container technology around image formats, runtimes, and distribution.
- [OCI Image layout](https://github.com/opencontainers/image-spec/blob/v1.1.0/image-layout.md): The OCI Image Layout specifies how to organize and store the image files on disk.


```shell
export IMAGE_SIGNED=docker.io/yizha1/net-monitor:v1
docker buildx create --use
docker buildx build . -f Dockerfile -o type=oci,dest=net-monitor.tar -t $IMAGE_SIGNED
mkdir net-monitor
tar -xf net-monitor.tar -C net-monitor
```

### Choose your KMS

Supported KMS:
- Azure Key Vault
- AWS Signer
- Alibaba Cloud Secrets manager
- Hashicorp Vault (alpha)

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
oras cp -r ./net-monitor:v1 --from-oci-layout $IMAGE_SIGNED
```

![image](https://github.com/user-attachments/assets/a919f28f-58ff-498b-816b-41e458f1732c)


### View from docker hub

Switch to [my docker hub](https://hub.docker.com/repositories/yizha1)


## Verify a container image in CI/CD pipelines.

CI/CD pipelines:
- GitHub actions
- Azure DevOps (ADO)
- FluxCD

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
             "x509.subject: CN=mycompany.io,O=Notary,L=Seattle,ST=WA,C=US"
         ]
     }
 ]
}
```

```shell
notation policy import mypolicy.json
notation policy show
notation verify $IMAGE_SIGNED
```

## Verify a container image before pulling to K8s

Knowledge points:
- OPA: Open Policy Agent, an open-source, general-purpose policy engine that enables fine-grained, declarative authorization and policy enforcement across various systems, including Kubernetes, microservices, CI/CD pipelines, and APIs.
- OPA Gatekeeper: A policy enforcement framework for Kubernetes that uses Open Policy Agent (OPA) to enforce fine-grained, customizable policies on Kubernetes resources. It works by intercepting API requests and evaluate them against policies before resources are created, modified or deleted.

Alternative:
- Azure policy + Ratify on Azure
- AKS Image integrity (Preview) on Azure
- Kyverno


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
helm install ratify ratify/ratify --atomic --namespace gatekeeper-system --set featureFlags.RATIFY_CERT_ROTATION=true --set-file notationCerts[0]="mycompany.io.crt"  --set notation.trustPolicies[0].registryScopes[0]="docker.io/yizha1/net-monitor" --set notation.trustPolicies[0].trustStores[0]=ca:notationCerts[0] --set notation.trustPolicies[0].trustedIdentities[0]="x509.subject: CN=mycompany.io\,O=Notary\,L=Seattle\,ST=WA\,C=US"
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



