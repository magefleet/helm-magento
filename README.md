# Magento2 Helm Chart

[![Version](https://img.shields.io/badge/version-3.2.2-blue.svg)](https://github.com/magefleet/helm/releases)
[![Magento](https://img.shields.io/badge/Magento-2.4.7--p6-orange.svg)](https://magento.com)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

This is a production ready helm chart for Magento2 and its services.
You can deploy immediately:
- Database (mariadb)
- Opensearch
- Redis
- RabbitMQ
- Varnish 
- Magento2
- Cert Manager for SSL Certificate with Lets Encrypt

The project includes best-of-breed charts from Bitnami, Elasticsearch and others to give maximum flexibility for deploying and
configuring the services.

Actually, due to [Bitnami Bombshell](https://medium.com/@talkimhi/bitnamis-august-28th-bombshell-the-end-of-free-container-images-as-we-know-them-74fe5cdfb882) we want to
switch all Bitnami containers to maintained homemade services

## Maintainers

This chart is maintained by [Francesco Oghabi](https://oghabi.it), a DevOps engineer specializing in Kubernetes and Magento deployments.

For questions, issues or contributions, please visit the [GitHub repository](https://github.com/magefleet/helm).

## Thanks To
Really Thanks  to phoenix media for their great work. They were the first to provide a Magento2 Helm Chart Open Source.
Without them this project would not exist.

### Difference with Phoenix Media Helm Chart
- Fixing Error 503 for ingress controller
- Use Longhorn for persistent storage and shareable media
- Production ready with environment values and secrets
- Direct configuration of TLS via Helm Chart
- Non-root permission for pods
- Attention to privilege escalation
- Avoid Bitnami and keep open source packages


## TL;DR


## Magento2 base image
The chart references [mine](https://oghabi.it) Magento2 Docker image. It consists of an
[Alpine nginx+PHP8.3 base image](https://github.com/francesco-oghabi/php-nginx), 
[Magento 2.4.7-p6](https://github.com/francesco-oghabi/magento2)
and a few Composer packages to add [build+deploy scripts](https://github.com/PHOENIX-MEDIA/magento2-cloud-build). The image is available on [DockerHub](https://hub.docker.com/r/phoenixmedia/magento2)
and the source code including Github Action is available in the [magento2-build repository](https://github.com/PHOENIX-MEDIA/magento2-build).

## Magento ECE-Tools
> ECE-Tools is a set of scripts and tools designed to manage and deploy Cloud projects.

Using [ECE-Tools](https://github.com/magento/ece-tools/) for building and deploying the Docker image reduces the amount
of custom scripts and also gives great flexibility to adjust process as needed.
It is recommended to get familiar with its [build and deploy mechanisms](https://devdocs.magento.com/cloud/project/magento-env-yaml.html).

It is required to review and adjust the [Magento Cloud environment variables](https://devdocs.magento.com/cloud/env/variables-cloud.html)
for the `magento` and `cronjob` deployment in the `values.yaml`:
- MAGENTO_CLOUD_ROUTES
- MAGENTO_CLOUD_RELATIONSHIPS
- MAGENTO_CLOUD_VARIABLES

Their values are simply base64 encoded JSON objects. To decode them either run `echo "<value>" | base64 -d` or use an
[online decoder](https://www.base64decode.org/) (beware when using sensitive data).

> **_Note:_** It is best practice to maintain sensitive values in a [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/) instead of keeping them in the `values.yaml`.

### Updating domains MAGENTO_CLOUD_* variables and values files
The values*.yaml files contain domain specific configurations. You will need to update a few lines the environment
variables of the magento, cronjob and xdebug (optional) workloads as well as in the ingress section:

```
secrets:
  credentials:
    data:
      MAGENTO_CLOUD_ROUTES: <base64-encoded-string-containing-your-domain>
      MAGENTO_CLOUD_ROUTES: <base64-encoded-string-containing-your-domain>

ingress:
  hosts:
    - name: <your-domain>
```


## Ingress

Before enabling the ingress make sure to configure `host.name` and the TLS certificate properly. When having
[cert-manager](https://cert-manager.io/docs/) installed set `ingress.certManager: true` to automatically generate a
certificate for the application.

If you prefer to skip Varnish for certain routes simply configure additional paths:

```
    paths:
    - path: "/"
      serviceName: varnish
      servicePort: 80
    - path: "/pub/media"
      serviceName: magento
      servicePort: 80
```

In case you want to protect the Magento backend by IP or BasicAuth we recommend to duplicate the `templates/ingress.yaml`
(e.g. to your Helm project root) and configure a second Ingress with proper annotations.

## Secrets

It is recommended to store sensitive information in a Kubernetes Secret (default name "general-secrets"). The Helm
Chart supports multiple ways to deploy secrets:

### Secrets in value file
Like in the default `values.yaml` sensitive information can be set directly in the YAML structure:

```
secrets:
  credentials:
    data:
      mariadb-password: topSecret
```

While this is okay for testing, it is not recommended to use this in production environments.

### Set sensitive information via CLI
In many CI/CD pipelines credentials are accessible by environment variables which can be easily passed via CLI:

```
helm install --set secrets.credentials.data.mariadb-password=$MYSQL_PASSWORD -f values.yaml magento2 .
```

### External Secrets Operator (ESO)
Credentials could be safely stored in a Vault. The [ESO](https://external-secrets.io/) can read credentials form all
major Vault providers, transform them and save them in a Kubernetes Secret. Even more it can automatically refresh them.

Support for ESO can be enabled simply by setting `secrets.externalSecrets.enabled=true`. The template for the SecretStore
allows flexible configuration of the preferred secret provider.

The `value.yaml` provides an example configuration for [Hashicorp Vault](https://external-secrets.io/v0.8.1/provider/hashicorp-vault/).
It also contains an ExternalSecret example for data and templates definition.
For more information see the ESO [documentation](https://external-secrets.io/v0.8.1/api/components/).

## Docker Registry Credentials

For pulling private Docker images, the chart supports automatic management of registry credentials via External Secrets Operator.

### Setup Docker Registry with External Secrets

1. **Store credentials in Vault:**

```bash
vault kv put magefleet/docker-registry \
  registry_url="docker.io" \
  registry_username="your-username" \
  registry_password="your-password" \
  registry_email="your-email@example.com"
```

2. **Enable in values.yaml:**

```yaml
global:
  imagePullSecrets:
    - name: docker-registry-secret

dockerRegistry:
  externalSecret:
    enabled: true
    name: docker-registry-credentials
    targetSecretName: docker-registry-secret
    secretStoreName: vault-backend
    secretStoreKind: ClusterSecretStore
    vaultPath: magefleet/docker-registry
    refreshInterval: 1h
```

The ExternalSecret will automatically create a Kubernetes secret of type `kubernetes.io/dockerconfigjson` that can be used to pull private images.

### Configuration Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `dockerRegistry.externalSecret.enabled` | Enable ExternalSecret for Docker registry credentials | `false` |
| `dockerRegistry.externalSecret.name` | Name of the ExternalSecret resource | `docker-registry-credentials` |
| `dockerRegistry.externalSecret.targetSecretName` | Name of the Kubernetes secret to create | `docker-registry-secret` |
| `dockerRegistry.externalSecret.secretStoreName` | Name of the SecretStore/ClusterSecretStore | `vault-backend` |
| `dockerRegistry.externalSecret.vaultPath` | Vault path where credentials are stored | `magefleet/docker-registry` |

### Verification

After deployment, verify the ExternalSecret is working:

```bash
# Check ExternalSecret status
kubectl get externalsecret docker-registry-credentials -n magento

# Verify the secret was created
kubectl get secret docker-registry-secret -n magento
``` 


## Persistence

For the media and var folder Magento usually requires an NFS share. Depending on the Kubernetes environment files shares
are available by referencing the correct storageClass.

In the `values.yaml` the `persistence` section and adjust it:

```
persistence:
  enabled: true
  name: magento-data
  #existingClaim:
  accessMode: ReadWriteMany
  size: 10Gi
  storageClassName: "files"
```

Existing [PVCs](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) can be also referenced
by using `existingClaim`.


## SMTP server
supervisord starts a simple Postfix MTA included in the [Alpine base image](https://github.com/PHOENIX-MEDIA/docker-nginx-php).
However, you should configure a mail relay which accepts mails for your Magento instance and is eligible to send emails for
the configured store email addresses.

Make sure to configure it for the `magento` and `cronjob` deployment:

```
    env:
    - name: RELAYHOST
      value: my.relayhost.com
    - name: SMTP_USE_TLS
      value: "true"
```

## imgproxy support
[imgproxy](https://imgproxy.net/) instantly resizes images and delivers it in an optimal format. This offloads resources
from the Magento pod and delivers images faster in PNG/WebP without additional effort.

To enable imgproxy in your deployment set `imgproxy.enabled: true` in your values file. Varnish will detect media image request
and will forward the request to imgproxy if available. The response will get cached response on a disk cache. For details check the
modified VCL in `values.yaml`. The relevant sections can be found by search for `x-img` in the VCL.

Resizing of product images happens on-the-fly once you enable the configuration in Magento the [URL format](https://experienceleague.adobe.com/docs/commerce-operations/configuration-guide/storage/remote-storage/remote-storage-image-resize.html?lang=en#configure-url-format-in-adobe-commerce).
This will append formatting instructions for width and height to the original product image URL. In addition to the clients accept headers
this information is used by imgproxy to deliver the image in the desired resolution.

> In case imgproxy can't serve the image the request will be gracefully forwarded to the Magento pod, which serves the 
configured placeholder image for requests to `media/catalog` and `media/wysiwyg`. The relevant VCL subrouting is `vcl_synth`.


## Helm deployment
The chart requires [Helm 3.x](https://helm.sh/) and has been tested with 3.9.0.
Make sure to adjust the `values.yaml` before deployment.

The Helm command is straight forward:

`helm upgrade -i -f  values_magento.yaml  --create-namespace -n magento magento .`

Deploying the whole Magento2 stack is complex operation and not unlikely when trying it the first time. We prefer to use
optional `--wait --timeout 15m` parameters in a deployment pipeline to see if the deployment was actually successful.

To deploy Magento to different environments (develop, staging, production) it is recommended to create a `values_*.yaml`
for each environment and tune the resource limits and configuration values of the services.



## Changelog

### [3.2.2] - 2025-11-09
- **Security**: Vault token no longer hardcoded in values.yaml - uses existingSecret instead
- **Added**: Support for vault.existingSecret to use external Vault token secret
- **Changed**: Updated artifacthub-repo.yml with security and prerelease annotations

### [3.2.1] - 2025-11-09
- **Changed**: Upgraded MariaDB subchart from 0.1.1 to 0.2.0 with External Secrets support
- **Added**: MariaDB subchart now supports existingSecret for external password management

### [3.2.0] - 2025-11-09
- **Changed**: External Secrets Operator now deploys to external-secrets-system namespace for better separation
- **Improved**: Better namespace organization and resource management

### [3.1.5] - 2025-11-07
- **Fixed**: Fixed imagePullSecrets template to support both string arrays and object arrays format
- **Fixed**: Prevent ArgoCD from pruning secrets created by External Secrets Operator

### [3.1.4] - 2025-11-06
- **Added**: Docker registry credentials support via ExternalSecret
- **Fixed**: Fixed ClusterIssuer namespace issue - removed namespace field from cluster-scoped resource
- **Changed**: Changed cert-manager default to disabled - recommended to install cluster-wide

### [3.1.3] - 2025-11-05
- **Fixed**: Docker registry support and cert-manager fixes

### [3.1.2] - 2025-11-04
- **Fixed**: GitHub Actions workflow improvements

### [3.1.1] - 2025-11-03
- **Added**: Documentation improvements and workflow fix

### [3.1.0] - 2025-11-02
- **Added**: Docker Registry External Secret support
- **Added**: GitHub Actions workflow and Artifact Hub metadata

### [3.0.0] - 2025-08-29
- **Added**: Longhorn support for persistent storage
- **Added**: CertManager for managing Let's Encrypt SSL certificates
- **Fixed**: Error 503 for ingress controller

