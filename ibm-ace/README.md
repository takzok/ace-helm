# IBM APP CONNECT ENTERPRISE

![ACE Logo](https://raw.githubusercontent.com/ot4i/ace-helm/master/appconnect_enterprise_logo.svg?sanitize=true)

## Introduction

IBM® App Connect Enterprise is a market-leading lightweight enterprise integration engine that offers a fast, simple way for systems and applications to communicate with each other. As a result, it can help you achieve business value, reduce IT complexity and save money. IBM App Connect Enterprise supports a range of integration choices, skills and interfaces to optimize the value of existing technology investments.

## Chart Details

By default this chart deploys a single IBM App Connect Enterprise for Developers integration server into a Kubernetes environment.

To deploy a production IBM App Connect Enterprise integration server, [check the installation instructions](README.md#installation-for-production).

## Prerequisites

* Kubernetes 1.9 or greater, with beta APIs enabled
* If persistence is enabled (see [configuration](#configuration)), then you either need to create a PersistentVolume, or specify a Storage Class if classes are defined in your cluster.
* If you have [pod security policy control](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#enabling-pod-security-policies) enabled, you must have a [PodSecurityPolicy](https://kubernetes.io/docs/concepts/policy/pod-security-policy/) that supports the following [securityContext](https://kubernetes.io/docs/concepts/policy/security-context/) settings:
  * capabilities:
    * CHOWN
    * DAC_OVERRIDE
    * FSETID
    * FOWNER
    * NET_RAW
    * SETGID
    * SETUID
    * SETFCAP
    * SETPCAP
    * NET_BIND_SERVICE
    * SYS_CHROOT
    * KILL
    * AUDIT_WRITE
  * allowPrivilegeEscalation: true
  * readOnlyRootFilesystem: false
  * runAsNonRoot: false
  * runAsUser: 0
  * privileged: false

* On RedHat OpenShift a [Security Context Constraint](https://blog.openshift.com/understanding-service-accounts-sccs/) should be used in place of a pod security policy. The recommended Security Context Constraint is shown below:
```
kind: SecurityContextConstraints
apiVersion: v1
metadata:
  name: ibm-ace-scc
allowPrivilegedContainer: true
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: RunAsAny
fsGroup:
  type: RunAsAny
supplementalGroups:
  type: RunAsAny
requiredDropCapabilities:
- MKNOD
allowedCapabilities:
- SETPCAP
- AUDIT_WRITE
- CHOWN
- NET_RAW
- DAC_OVERRIDE
- FOWNER
- FSETID
- KILL
- SETUID
- SETGID
- NET_BIND_SERVICE
- SYS_CHROOT
- SETFCAP  
forbiddenSysctls:
- '*'    
```
* **Note**: If you are deploying to an IBM Cloud Private environment that does not support these security settings by default. Follow these [instructions](https://www.ibm.com/support/knowledgecenter/SSBS6K_3.1.0/app_center/nd_helm.html) to enable your deployment.

* If you are using SELinux you must meet the [MQ requirements](https://www-01.ibm.com/support/docview.wss?uid=swg21714191)

To separate secrets from the Helm release a secret can be pre-installed with the following shape and referenced from the Helm chart with the `configurationSecret` value:
```
apiVersion: v1
kind: Secret
metadata:
  name: <secretName>
type: Opaque
data:
  adminPassword:
  appPassword:
  keystoreCert-<alias>:
  keystoreKey-<alias>:
  keystorePass-<alias>:
  keystorePassword:
  mqsc:
  odbcini:
  policy:
  policyDescriptor:
  serverconf:
  setdbparms:
  truststoreCert-<alias>:
  truststorePassword:
```

The following table describes the secret keys:

| Key                             | Description                                                        |
| ------------------------------- | ------------------------------------------------------------------ |
| `adminPassword`                 | MQ Developer defaults - administrator password                     |
| `appPassword`                   | MQ Developer defaults - app password                               |
| `keystorePass-{keyname}`        | The passphrase for the private key being imported, if there is one |
| `keystoreKey-{keyname}`         | Multi-line value containing the private key in PEM format                                      |
| `keystoreCert-{keyname}`        | Multi-line value containing the certificate in PEM format                                      |
| `keystorePassword`              | A password to set for the Integration Server's keystore            |
| `mqsc`                          | Multi-line value containing an mqsc file to run against the Queue Manager |
| `odbcini`                       | Multi-line value containing an odbc.ini file for the Integration Server to define any ODBC data connections |
| `policy`                        | Multi-line value containing a policy to apply                      |
| `policyDescriptor`              | Multi-line value containing the policy descriptor file             |
| `serverconf`                    | Multi-line value containing a server.conf.yaml                                               |
| `setdbparms`                    | Multi-line value containing the `{ResourceName} {UserId} {Password}` to pass to [mqsisetdbparms command](https://www.ibm.com/support/knowledgecenter/en/SSTTDS_11.0.0/com.ibm.etools.mft.doc/an09155_.htm) |
| `truststoreCert-{certname}`     | Multi-line value containing the trust certificate in PEM format    |
| `truststorePassword`            | A password to set for the Integration Server's truststore          |

A second secret may also be pre-installed with the following shape for a release with the name `{{ .RELEASE.NAME }}-ibm-ace`:
```
apiVersion: v1
kind: Secret
metadata:
  name: {{ .RELEASE.NAME }}-ibm-ace
type: Opaque
data:
  adminusers:
  viewerusers:
```

The following table describes the secret keys:

| Key                             | Description                                                        |
| ------------------------------- | ------------------------------------------------------------------ |
| `adminusers`                    | Multi-line value containing the `{UserName} {Password}` to pass to [mqsiwebuseradmin command](https://www.ibm.com/support/knowledgecenter/en/SSTTDS_11.0.0/com.ibm.etools.mft.doc/bn28490_.htm) to create viewer users with READ, WRITE, EXECUTE access to the server |
| `viewerusers`                   | Multi-line value containing the `{UserName} {Password}` to pass to [mqsiwebuseradmin command](https://www.ibm.com/support/knowledgecenter/en/SSTTDS_11.0.0/com.ibm.etools.mft.doc/bn28490_.htm) to create admin users with READ access to the server |

Further instructions and helper scripts are provided in the `scripts` directory.

## Resources Required

This chart uses the following resources by default:

* 1 CPU core
* 1 GiB memory without MQ
* 2 GiB memory with MQ

See the [configuration](#configuration) section for how to configure these values.

## Installing the Chart

First, upload your ACE and/or ACE & MQ images to a Docker registry. You will have to indicate the location of these images in the chart installation.
To build your ACE / ACE & MQ images, read the [instructions in the `ace-docker` repository](https://github.com/ot4i/ace-docker).

If using a private Docker registry, an image pull secret needs to be created before installing the chart. Supply the name of the secret as the value for `image.pullSecret`.

**By default this chart installs IBM App Connect Enterprise for Developers (without MQ).**

### ACE for Developers installation

1. ACE for Developers only:

To install the chart with the release name `ace-dev`:
```
helm install --name ace-dev ibm-ace --set license=accept --set image.repository.aceonly={docker-repo/ace-image-name} --set image.tag={image-tag}
```

1. ACE & MQ Advanced for Developers:

To install the chart with the release name `ace-mq-dev`:
```
helm install --name ace-mq-dev ibm-ace --set license=accept --set image.repository.acemq={docker-repo/ace-mq-image-name} --set image.tag={image-tag} --set queueManagerEnabled=true
```

### ACE installation

1. ACE only:

To install the chart with the release name `ace`:
```
helm install --name ace ibm-ace --set license=accept --set image.repository.aceonly={docker-repo/ace-image-name} --set image.tag={image-tag} --set devEdition=false
```

1. ACE & MQ Advanced:

To install the chart with the release name `ace-mq`:
```
helm install --name ace-mq ibm-ace --set license=accept --set image.repository.acemq={docker-repo/ace-mq-image-name} --set image.tag={image-tag} --set queueManagerEnabled=true --set devEdition=false
```

- **Note:** If installing to a pre-production or production environment, also set `productionDeployment` to `true` on the installation command.

To deploy an IBM App Connect Enterprise server on the Kubernetes cluster this command:
- accepts the [IBM App Connect Enterprise license](../LICENSE)
- sets the docker repository + image name for `aceonly` or `acemq`
- sets the image tag
- sets whether MQ is to be enabled with `queueManagerEnabled`
- sets if the image used is a Developers edition with `devEdition`.
The [configuration](#configuration) section lists the parameters that can be configured during installation.

- **Tip**: See all the resources deployed by the chart using `kubectl get all -l release={release-name}`

## Installing a sample image

If you have built your own docker image following the [docker instructions in `ace-docker`](https://github.com/ot4i/ace-docker) and you want to test installing the [`sample` image](https://github.com/ot4i/ace-docker/sample), we have provided a file `samplevalues.yaml` that has all the values needed to run the sample image.

First run the `generateSecrets.sh` script using the configuration files in scripts/sample-configuration-files to generate the required secrets separate from the Helm release:

```
cd scripts
cp ./sample-configuration-files/* ./
./generateSecrets.sh my-secret {{ .Release.Name }}
```

To install the chart using ACE only and the sample values:

```
helm install --name ace-dev ibm-ace --set license=accept -f=./ibm-ace/samplevalues.yaml --set image.repository.aceonly={docker-repo/ace-image-name} --set image.tag={image-tag}
```

To install the chart using ACE & MQ and the sample values:

```
helm install --name ace-mq-dev ibm-ace --set license=accept -f=./ibm-ace/samplevalues.yaml --set image.repository.acemq={docker-repo/ace-mq-image-name} --set image.tag={image-tag} --set queueManagerEnabled=true
```


## Uninstalling the Chart

To uninstall/delete a `rel1` release:

```
helm delete --purge rel1
```

The command removes all the Kubernetes components associated with the chart, except any Persistent Volume Claims (PVCs). This is the default behaviour of Kubernetes, and ensures that valuable data is not deleted.

## Extra features in IBM Cloud Private: IBM ACE Dashboard

When installing in IBM Cloud Private you also have access to IBM ACE Dashboard, which provides a UI to manage Integration Servers. You can install the IBM ACE Dashboard for Developers from ICP catalog, search for `ibm-ace-dashboard-dev`. If you have loaded the ACE cloud pack then you can also install the production IBM ACE Dashboard `ibm-ace-dashboard-prod`.

Once the Dashboard is installed, any ACE Integration Servers or ACE & MQ integration servers that are deployed using the installation instructions here will show up in the Dashboard UI.
For these servers to be able to be inspected in the Dashboard, an extra user needs to be added to the installation configuration:
Add an entry for a user with read access named `ibm-ace-dashboard-viewer` to `integrationServer.viewerusers` so the Dashboard UI can display information from the server.


## Configuration
The following table lists the configurable parameters of the `ibm-ace` chart and their default values.

| Parameter                        | Description                                     | Default                                                    |
| -------------------------------- | ----------------------------------------------- | ---------------------------------------------------------- |
| `license`                        | Set to `accept` to accept the terms of the IBM license  | `Not accepted`                                     |
| `productionDeployment`           | Boolean toggle for whether ACE server is being run in a production environment or a pre-production environment | `false`  |
| `devEdition`                     | Boolean toggle for whether ACE for Developers or ACE is being used | `true`                                  |
| `queueManagerEnabled`            | Boolean toggle for whether to run a StatefulSet with an MQ Queue Manager associated with the ACE Integration Server, or a Deployment without an MQ Queue Manager| `false` |
| `image.repository.aceonly`       | The repository and name for the Docker image with ACE only | `ace-only`                                      |
| `image.repository.acemq`         | The repository and name for the Docker image with ACE with MQ | `ace-mq`                                     |
| `image.tag`                      | Image tag                                       | `latest`                                                   |
| `image.pullPolicy`               | Image pull policy                               | `IfNotPresent`                                             |
| `image.pullSecret`               | Image pull secret, if you are using a private Docker registry | `nil`                                        |
| `arch`                           | Architecture scheduling preference for worker node (only amd64 supported) - read only | `amd64`              |
| `fsGroupGid`                     | File system group ID for volumes that support ownership management | `nil`                                   |
| `persistence.enabled`            | Use Persistent Volumes for all defined volumes  | `true`                                                     |
| `persistence.useDynamicProvisioning`| Use dynamic provisioning (storage classes) for all volumes | `true`                                       |
| `dataPVC.name`                   | Suffix for the Persistent Volume Claim name     | `data`                                                     |
| `dataPVC.storageClassName`       | Storage class of volume for main MQ data (under /var/mqm) | `nil`                                            |
| `dataPVC.size`                   | Size of volume for main MQ data (under /var/mqm) | `2Gi`                                                     |
| `service.type`                   | Kubernetes service type exposing ports          | `NodePort`                                                 |
| `service.webuiPort`              | Web UI port number - read only                  | `7600`                                                     |
| `service.serverlistenerPort`     | Http server listener port number - read only    | `7800`                                                     |
| `service.serverlistenerTLSPort`  | Https server listener port number - read only   | `7843`                                                     |
| `service.iP`                     | This is a hostname/IP that the nodeport is connected to i.e. a workers IP    | `nil`               |
| `aceonly.resources.limits.cpu`        | Kubernetes CPU limit for the container      | `1`                                                       |
| `aceonly.resources.limits.memory`     | Kubernetes memory limit for the container   | `1024Mi`                                                  |
| `aceonly.resources.requests.cpu`      | Kubernetes CPU request for the container    | `1`                                                       |
| `aceonly.resources.requests.memory`   | Kubernetes memory request for the container | `1024Mi`                                                  |
| `acemq.resources.limits.cpu`      | Kubernetes CPU limit for the container      | `1`                                                           |
| `acemq.resources.limits.memory`   | Kubernetes memory limit for the container   | `2048Mi`                                                      |
| `acemq.resources.requests.cpu`    | Kubernetes CPU request for the container    | `1`                                                           |
| `acemq.resources.requests.memory` | Kubernetes memory request for the container | `2048Mi`                                                      |
| `replicaCount`                   | Set how many replicas to run | `3`               |
| `queueManager.name`              | MQ Queue Manager name                           | Helm release name                                          |
| `integrationServer.name`         | ACE Integration Server name                     | Helm release name                                          |
| `integrationServer.keystoreKeyNames`     | Comma separated list of keystore aliases in the configuration secret | `nil`                         |
| `integrationServer.truststoreCertNames`  | Comma separated list of truststore aliases in the configuration secret | `nil`                       |
| `configurationSecret`            | The name of the secret to create or to use that contains the server configuration | `nil`                    |
| `log.format`                     | Error log format on container's console. Either `json` or `basic` | `json`                                   |
| `metrics.enabled`                | Enable Prometheus metrics for the Queue Manager and Integration Server | `true`                              |
| `livenessProbe.initialDelaySeconds` | The initial delay before starting the liveness probe. Useful for slower systems that take longer to start the Queue Manager |	`180` |
| `livenessProbe.periodSeconds`    | How often to run the probe                      | `10`                                                       |
| `livenessProbe.timeoutSeconds`   | Number of seconds after which the probe times out | `5`                                                      |
| `livenessProbe.failureThreshold` | Minimum consecutive failures for the probe to be considered failed after having succeeded | `1`              |
| `readinessProbe.initialDelaySeconds`| The initial delay before starting the readiness probe | `10`                                              |
| `readinessProbe.periodSeconds`   | How often to run the probe                      | `5`                                                        |
| `readinessProbe.timeoutSeconds`  | Number of seconds after which the probe times out | `3`                                                      |
| `readinessProbe.failureThreshold`| Minimum consecutive failures for the probe to be considered failed after having succeeded | `1`              |

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`.

Alternatively, a YAML file that specifies the values for the parameters can be provided while installing the chart.

- **Tip**: You can use the default [values.yaml](values.yaml)

## Storage

When Queue Manager is enabled, the chart mounts a [Persistent Volume](http://kubernetes.io/docs/user-guide/persistent-volumes/) for the storage of MQ configuration data and messages.  By using a Persistent Volume based on network-attached storage, Kubernetes can re-schedule the MQ server onto a different worker node. You should not use "hostPath" or "local" volumes, because this will not allow moving between nodes. The default size of the persistent volume claim is 2Gi.

Performance requirements will vary widely based on workload, but as a guideline, use a Storage Class which allows for at least 200 IOPS (based on 16 KB block size with a 50/50 read/write mix).

For volumes that support onwership management, specify the group ID of the group owning the persistent volumes' file systems using the `fsGroupGid` parameter.

## Logging

The `log.format` value controls whether the format of the output logs is:
- basic: Human readable format intended for use in development, such as when viewing through `kubectl logs`
- json: Provides more detailed information for viewing through Kibana

## Limitations

This Chart can run only on amd64 architecture type.

## Useful Links

[View the IBM App Connect Enterprise Dockerfile repository on Github](https://github.com/ot4i/ace-docker)

[View the Official IBM App Connect Enterprise for Developers Docker Image in Docker Hub](https://hub.docker.com/r/ibmcom/ace/)

[Learn more about IBM App Connect Enterprise](https://www.ibm.com/support/knowledgecenter/en/SSTTDS_11.0.0/com.ibm.ace.home.doc/help_home.htm)

[Learn more about IBM App Connect Enterprise and Docker](https://www.ibm.com/support/knowledgecenter/en/SSTTDS_11.0.0/com.ibm.etools.mft.doc/bz91300_.htm)

[Learn more about IBM App Connect Enterprise and Lightweight Integration](https://ibm.biz/LightweightIntegrationLinks)

_Copyright IBM Corporation 2018. All Rights Reserved._

_The IBM App Connect Enterprise logo is copyright IBM. You will not use the IBM App Connect Enterprise logo in any way that would diminish the IBM or IBM App Connect Enterprise image. IBM reserves the right to end your privilege to use the logo at any time in the future at our sole discretion. Any use of the IBM App Connect Enterprise logo affirms that you agree to adhere to these conditions._
