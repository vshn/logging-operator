# Custom Resource Definitions

This document contains the detailed information about the CRDs logging-operator uses.

Available CRDs:
- [loggings.logging.banzaicloud.io](/config/crd/bases/logging.banzaicloud.io_loggings.yaml)
- [outputs.logging.banzaicloud.io](/config/crd/bases/logging.banzaicloud.io_outputs.yaml)
- [flows.logging.banzaicloud.io](/config/crd/bases/logging.banzaicloud.io_flows.yaml)
- [clusteroutputs.logging.banzaicloud.io](/config/crd/bases/logging.banzaicloud.io_clusteroutputs.yaml)
- [clusterflows.logging.banzaicloud.io](/config/crd/bases/logging.banzaicloud.io_clusterflows.yaml)

> You can find example yamls  [here](/docs/examples)

## loggings

Logging resource define a logging infrastructure for your cluster. You can define **one** or **more** `logging` resource. This resource holds together a `logging pipeline`. It is responsible to deploy `fluentd` and `fluent-bit` on the cluster. It declares a `controlNamespace` and `watchNamespaces` if applicable.

> Note: The `logging` resources are referenced by `loggingRef`. If you setup multiple `logging flow` you have to reference other objects to this field. This can happen if you want to run multiple fluentd with separated configuration.

You can install `logging` resource via [Helm chart](/charts/logging-operator-logging) with built-in TLS generation.

### Namespace separation
A `logging pipeline` consist two type of resources.
- `Namespaced` resources: `Flow`, `Output`
- `Global` resources: `ClusterFlow`, `ClusterOutput`

The `namespaced` resources only effective in their **own** namespace. `Global` resources are operate **cluster wide**. 

> You can only create `ClusterFlow` and `ClusterOutput` in the `controlNamespace`. It **MUST** be a **protected** namespace that only **administrators** have access.

Create a namespace for logging
```bash
kubectl create ns logging
```

**`logging` plain example** 
```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Logging
metadata:
  name: default-logging-simple
  namespace: logging
spec:
  fluentd: {}
  fluentbit: {}
  controlNamespace: logging
```

**`logging` with filtered namespaces** 
```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Logging
metadata:
  name: default-logging-namespaced
  namespace: logging
spec:
  fluentd: {}
  fluentbit: {}
  controlNamespace: logging
  watchNamespaces: ["prod", "test"]
```

### Logging parameters
| Name                    | Type           | Default | Description                                                             |
|-------------------------|----------------|---------|-------------------------------------------------------------------------|
| loggingRef              | string         | ""      | Reference name of the logging deployment                                |
| flowConfigCheckDisabled | bool           | False   | Disable configuration check before deploy                               |
| flowConfigOverride      | string         | ""      | Use static configuration instead of generated config.                   |  
| fluentbit               | [FluentbitSpec](#Fluent-bit-Spec) | {}      | Fluent-bit configurations                                               |
| fluentd                 | [FluentdSpec](#Fluentd-Spec)   | {}      | Fluentd configurations                                                  |
| watchNamespaces         | []string       | ""      | Limit namespaces from where to read Flow and Output specs               |
| controlNamespace        | string         | ""      | Control namespace that contains ClusterOutput and ClusterFlow resources |

#### Fluentd Spec

You can customize the `fluentd` statefulset with the following parameters.

| Name                    | Type           | Default | Description                                                             |
|-------------------------|----------------|---------|-------------------------------------------------------------------------|
| annotations | map[string]string | {} | Extra annotations to Kubernetes resource|
| tls | [TLS](#TLS-Spec) | {} | Configure TLS settings|
| image | [ImageSpec](#Image-Spec) | {} | Fluentd image override |
| fluentdPvcSpec | [PersistentVolumeClaimSpec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#persistentvolumeclaimspec-v1-core) | {} | FLuentd PVC spec to mount persistent volume for Buffer |
| disablePvc | bool | false | Disable PVC binding |
| volumeModImage | [ImageSpec](#Image-Spec) | {} | Volume modifier image override |
| configReloaderImage | [ImageSpec](#Image-Spec) | {} | Config reloader image override |
| resources | [ResourceRequirements](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#resourcerequirements-v1-core) | {} | Resource requirements and limits |

**`logging` with custom fluentd pvc** 
```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Logging
metadata:
  name: default-logging-simple
  namespace: logging
spec:
  fluentd: 
    fluentdPvcSpec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 40Gi
        storageClassName: fast
  fluentbit: {}
  controlNamespace: logging
```

#### Fluent-bit Spec
| Name                    | Type           | Default | Description                                                             |
|-------------------------|----------------|---------|-------------------------------------------------------------------------|
| annotations | map[string]string | {} | Extra annotations to Kubernetes resource|
| tls | [TLS](#TLS-Spec) | {} | Configure TLS settings|
| image | [ImageSpec](#Image-Spec) | {} | Fluentd image override |
| resources | [ResourceRequirements](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#resourcerequirements-v1-core) | {} | Resource requirements and limits |
| targetHost | string | *Fluentd host* | Hostname to send the logs forward |
| targetPort | int | *Fluentd port* |  Port to send the logs forward |
| parser | string | cri | Change fluent-bit input parse configuration. [Available parsers](https://github.com/fluent/fluent-bit/blob/master/conf/parsers.conf)  |
| position_db |  [KubernetesStorage](#KubernetesStorage) | nil | Add position db storage support |
  
**`logging` with custom fluent-bit annotations** 
```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Logging
metadata:
  name: default-logging-simple
  namespace: logging
spec:
  fluentd: {}
  fluentbit:
    annotations:
      my-annotations/enable: true
  controlNamespace: logging
```

#### Image Spec

Override default images

| Name                    | Type           | Default | Description |
|-------------------------|----------------|---------|-------------|
| repository | string | "" | Image repository |
| tag | string | "" | Image tag |
| pullPolicy | string | "" | Always, IfNotPresent, Never |

**`logging` with custom fluentd image** 
```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Logging
metadata:
  name: default-logging-simple
  namespace: logging
spec:
  fluentd: 
    image:
      repository: banzaicloud/fluentd
      tag: v1.6.3-alpine-3
      pullPolicy: IfNotPresent
  fluentbit: {}
  controlNamespace: logging
```

#### TLS Spec	

Define TLS certificate secret

| Name                    | Type           | Default | Description |
|-------------------------|----------------|---------|-------------|
| enabled | string | "" | Image repository |
| secretName | string | "" | Kubernetes secret that contains: **tls.crt, tls.key, ca.crt** |
| sharedKey | string | "" | Shared secret for fluentd authentication |


**`logging` setup with TLS**
```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Logging
metadata:
  name: default-logging-tls
  namespace: logging
spec:
  fluentd:
    disablePvc: true
    tls:
      enabled: true
      secretName: fluentd-tls
      sharedKey: asdadas
  fluentbit:
    tls:
      enabled: true
      secretName: fluentbit-tls
      sharedKey: asdadas
  controlNamespace: logging

```

#### KubernetesStorage

Define Kubernetes storage

| Name      | Type | Default | Description |
|-----------|------|---------|-------------|
| host_path | [HostPathVolumeSource](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.12/#hostpathvolumesource-v1-core) | - | Represents a host path mapped into a pod. |


## outputs, clusteroutputs

Outputs are the final stage for a `logging flow`. You can define multiple `outputs` and attach them to multiple `flows`.

> Note: `Flow` can be connected to `Output` and `ClusterOutput` but `ClusterFlow` is only attachable to `ClusterOutput`.

### Defining outputs

The supported `Output` plugins are documented [here](./plugins/outputs)

| Name                    | Type              | Default | Description |
|-------------------------|-------------------|---------|-------------|
| **Output Definitions** | [Output](./plugins/outputs) | nil | Named output definitions |
| loggingRef | string | "" | Specified `logging` resource reference to connect `Output` and `ClusterOutput` to |


**`output` s3 example**
```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Output
metadata:
  name: s3-output-sample
spec:
  s3:
    aws_key_id:
      valueFrom:
        secretKeyRef:
          name: s3-secret
          key: awsAccessKeyId
          namespace: default
    aws_sec_key:
      valueFrom:
        secretKeyRef:
          name: s3-secret
          key: awsSecretAccesKey
          namespace: default
    s3_bucket: example-logging-bucket
    s3_region: eu-west-1
    path: logs/${tag}/%Y/%m/%d/
    buffer:
      timekey: 1m
      timekey_wait: 10s
      timekey_use_utc: true
```

## flows, clusterflows

Flows define a `logging flow` that defines the `filters` and `outputs`.

> `Flow` resources are `namespaced`, the `selector` only select `Pod` logs within namespace.
> `ClusterFlow` select logs from **ALL** namespace.

### Parameters
| Name                    | Type              | Default | Description |
|-------------------------|-------------------|---------|-------------|
| selectors               | map[string]string | {}      | Kubernetes label selectors for the log. |
| filters                 | [][Filter](./plugins/filters)          | []      | List of applied [filter](./plugins/filters).  |
| loggingRef              | string | "" | Specified `logging` resource reference to connect `FLow` and `ClusterFlow` to |
| outputRefs              | []string | [] | List of [Outputs](#Defining-outputs) or [ClusterOutputs](#Defining-outputs) names |

*`flow` example with filters and output in the `default` namespace*
```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Flow
metadata:
  name: flow-sample
  namespace: default
spec:
  filters:
    - parse:
        key_name: log
        remove_key_name_field: true
        parsers:
          - type: nginx
    - tag_normaliser:
        format: ${namespace_name}.${pod_name}.${container_name}
  outputRefs:
    - s3-output
  selectors:
    app: nginx
```