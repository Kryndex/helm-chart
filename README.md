# StorageOS

[StorageoOS](https://storageos.com) is a software-based storage platform designed for cloud-native applications.  By
deploying StorageOS on your Kubernetes cluster, local storage from cluster node is aggregated into a distributed pool,
and persistent volumes created from it using the native Kubernetes volume driver are available instantly to pods
wherever they move in the cluster.

Features such as replication, encryption and caching help protect data and maximise performance.

## TL;DR

```console
$ git clone https://github.com/storageos/helm-chart.git storageos
$ cd storageos
$ helm install .
```

## Prerequisites

- Kubernetes 1.8+ with Beta APIs enabled
- Kubernetes must be configured to allow:
    - Privileged mode containers (enabled by default)
    - Feature gate: MountPropagation=true.  This can be done by appending `--feature-gates MountPropagation=true` to the
      kube-apiserver start command.

## Installing the Chart

To install the chart with the release name `my-release`:

```console
$ helm install --name my-release .
```

The command deploys StorageOS on the Kubernetes cluster in the default configuration. The [configuration](#configuration)
section lists the parameters that can be configured during installation.

> **Tip**: List all releases using `helm list`

## Uninstalling the Chart

To uninstall/delete the `my-release` deployment:

```console
$ helm delete my-release
```

The command removes all the Kubernetes components associated with the chart and deletes the release.

## Configuration

The `cluster.join` parameter must be set to a vaild join string.  The join string helps bootstrap a new cluster and
provides instructions to nodes joining an existing cluster.  Typically the join string should be composed of a cluster
id and the internal service address of the StorageOS api, for example:

```console
$ helm install . --name my-release \
    --set cluster.join=74e8b44d-b1df-11e7-b0b3-42010a9a00b2,http://storageos:5705
```

The first item in the list can be a cluster id as above, or a hostname or ip address of a single node in the cluster.

A cluster id can be generated by running the `storageos cluster create` CLI command.  The CLI is available to download
from the [Github repository](https://github.com/storageos/go/-cli).

Alternatively, set the first item in the list to be the ip address or hostname of one of the cluster nodes.  This node
will boostrap the cluster when StorageOS is started for the first time on it.  It only serves a special purpose until
the cluster has initialised.

The remaining items in the join list should be one or more hostnames or ip addresses for new node to join to.  The api
service endpoint can be used here (`http://storageos:5705`) to avoid specifying individual hostnames or addresses.

> **Tip**: Future releases will remove the requirement to specify `cluster.join` and instead the [discovery service](https://github.com/storageos/discovery)
will run as part of the deployment.

The following tables lists the configurable parameters of the StorageOS chart and their default values.

Parameter | Description | Default
--------- | ----------- | -------
`cluster.join` | The cluster join string.  See [cluster discovery](https://docs.storageos.com/docs/install/prerequisites/clusterdiscovery) documentation for details.
`image.repository` | StorageOS container image repository | `storageos/node`
`image.tag` | StorageOS container image tag | `latest`
`image.pullPolicy` | StorageOS container image pull policy | `IfNotPresent`
`storageclass.name` | StorageOS storage class name | `fast`
`storageclass.pool` | Default storage pool for storage class | `default`
`storageclass.fsType` | Default filesystem type for storage class | `ext4`
`api.secretName` | Name of the secret used for storing api location and credentials | `storageos-api`
`api.secretNamespace` | Namespace of the secret used for storing api location and credentials | `default`
`api.address` | Hostname or IP address of the external StorageOS api endpoint.  This must be accessible from the Kubernetes master. | `http://storageos:5705`
`api.username` | Username to authenticate to the StorageOS api with | `storageos`
`api.password` | Password to authenticate to the StorageOS api with | `storageos`
`service.name` | Name of the StorageOS api service | `storageos`
`service.type` | Type of service to create | `LoadBalancer`
`service.externalIPs` | Service external IP addresses | `[]`
`service.loadBalancerIP` | IP address to assign to load balancer (if supported) | `""`
`service.loadBalancerSourceRanges` | list of IP CIDRs allowed access to load balancer (if supported) | `[]`
`service.externalPort` | External service port | `15705`
`service.internalPort` | Internal service port | `5705`
`ingress.enabled` | If true, Ingress will be created | `false`
`ingress.annotations` | Ingress annotations | `{}`
`ingress.hosts` | Ingress hostnames | `[]`
`ingress.tls` | Ingress TLS configuration (YAML) | `[]`
`resources` | Pod resource requests & limits | `{}`


Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`. For example,

```console
$ helm install . --name my-release \
    --set cluster.firstNode=gke-test01-node-default-pool-478bef52-wljs
```

Alternatively, a YAML file that specifies the values for the above parameters can be provided while installing the chart. For example,

```console
$ helm install . --name my-release -f values.yaml
```

> **Tip**: You can use the default [values.yaml](values.yaml)

### Ingress TLS
If your cluster allows automatic creation/retrieval of TLS certificates (e.g. [kube-lego](https://github.com/jetstack/kube-lego)), please refer to the documentation for that mechanism.

To manually configure TLS, first create/retrieve a key & certificate pair for the address(es) you wish to protect. Then create a TLS secret in the namespace:

```console
kubectl create secret tls storageos-tls --cert=path/to/tls.cert --key=path/to/tls.key
```

Include the secret's name, along with the desired hostnames, in the Ingress TLS section of your custom `values.yaml` file:

```yaml
server:
  ingress:
    ## If true, StorageOS Ingress will be created
    ##
    enabled: true

    ## StorageOS Ingress hostnames
    ## Must be provided if Ingress is enabled
    ##
    hosts:
      - storageos.domain.com

    ## StorageOS Ingress TLS configuration
    ## Secrets must be manually created in the namespace
    ##
    tls:
      - secretName: storageos-tls
        hosts:
          - storageos.domain.com
```