---
title: Configure Fluentd
shorttitle: Fluentd
weight: 40
---

You can configure the Fluentd deployment via the **fluentd** section of the {{% xref "/docs/one-eye/logging-operator/configuration/logging.md" %}}. This page shows some examples on configuring Fluentd. For the detailed list of available parameters, see {{% xref "/docs/one-eye/logging-operator/configuration/crds/v1beta1/fluentd_types.md" %}}.

## Custom pvc volume for Fluentd buffers

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Logging
metadata:
  name: default-logging-simple
spec:
  fluentd:
    bufferStorageVolume:
      pvc:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 40Gi
          storageClassName: fast
          volumeMode: Filesystem
  fluentbit: {}
  controlNamespace: logging
```

## Custom Fluentd hostPath volume for buffers

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Logging
metadata:
  name: default-logging-simple
spec:
  fluentd:
    disablePvc: true
    bufferStorageVolume:
      hostPath:
        path: "" # leave it empty to automatically generate: /opt/logging-operator/default-logging-simple/default-logging-simple-fluentd-buffer
  fluentbit: {}
  controlNamespace: logging
```

## FluentOutLogrotate

The following snippet redirects Fluentd's stdout to a file and configures rotation settings.

This is important to avoid Fluentd getting into a ripple effect when there is an error and the error message gets back to the system as a log message, which generates another error, and so on.

Default settings configured by the operator:

```yaml
spec:
  fluentd:
    fluentOutLogrotate:
      enabled: true
      path: /fluentd/log/out
      age: 10
      size: 10485760
```

Disabling it and write to stdout (not recommended):

```yaml
spec:
  fluentd:
    fluentOutLogrotate:
      enabled: false
```

## Scaling

You can scale the Fluentd deployment by changing the number of replicas in the **fluentd** section of the {{% xref "/docs/one-eye/logging-operator/configuration/logging.md" %}}. For example:

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Logging
metadata:
  name: default-logging-simple
spec:
  fluentd:
    scaling:
      replicas: 3
  fluentbit: {}
  controlNamespace: logging
```

### Graceful draining

While you can scale down the Fluentd deployment by decreasing the number of replicas in the **fluentd** section of the {{% xref "/docs/one-eye/logging-operator/configuration/logging.md" %}}, it won't automatically be graceful, as the controller will stop the extra replica pods without waiting for any remaining buffers to be flushed.
You can enable graceful draining in the **scaling** subsection:

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Logging
metadata:
  name: default-logging-simple
spec:
  fluentd:
    scaling:
      drain:
        enabled: true
  fluentbit: {}
  controlNamespace: logging
```

When graceful draining is enabled, the operator starts drainer jobs for any undrained volumes.
The drainer job flushes any remaining buffers before terminating, and the operator marks the associated volume (the PVC, actually) as *drained* until it gets used again.
The drainer job has a template very similar to that of the Fluentd deployment with the addition of a sidecar container that oversees the buffers and signals Fluentd to terminate when all buffers are gone.
Pods created by the job are labeled as not to receive any further logs, thus buffers will clear out eventually.

If you want, you can specify a custom drainer job sidecar image in the **drain** subsection:

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Logging
metadata:
  name: default-logging-simple
spec:
  fluentd:
    scaling:
      drain:
        enabled: true
        image:
          repository: ghcr.io/banzaicloud/fluentd-drain-watch
          tag: latest
  fluentbit: {}
  controlNamespace: logging
```

In addition to the drainer job, the operator also creates a placeholder pod with the same name as the terminated pod of the Fluentd deployment to keep the deployment from recreating that pod which would result in concurrent access of the volume.
The placeholder pod just runs a pause container, and goes away as soon as the job has finished successfully or the deployment is scaled back up and explicitly flushing the buffers is no longer necessary because the newly created replica will take care of processing them.

You can mark volumes that should be ignored by the drain logic by adding the label `logging.banzaicloud.io/drain: no` to the PVC.

## Probe

A [Probe](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) is a diagnostic performed periodically by the kubelet on a Container. To perform a diagnostic, the kubelet calls a Handler implemented by the Container. You can configure a probe for Fluentd in the **livenessProbe** section of the {{% xref "/docs/one-eye/logging-operator/configuration/logging.md" %}}. For example:

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Logging
metadata:
  name: default-logging-simple
spec:
  fluentd:
    livenessProbe:
      periodSeconds: 60
      initialDelaySeconds: 600
      exec:
        command:
        - "/bin/sh"
        - "-c"
        - >
          LIVENESS_THRESHOLD_SECONDS=${LIVENESS_THRESHOLD_SECONDS:-300};
          if [ ! -e /buffers ];
          then
            exit 1;
          fi;
          touch -d "${LIVENESS_THRESHOLD_SECONDS} seconds ago" /tmp/marker-liveness;
          if [ -z "$(find /buffers -type d -newer /tmp/marker-liveness -print -quit)" ];
          then
            exit 1;
          fi;
  fluentbit: {}
  controlNamespace: logging
```

You can use the following parameters:

| Name                    | Type           | Default | Description |
|-------------------------|----------------|---------|-------------|
| initialDelaySeconds | int | 0 | Number of seconds after the container has started before liveness probes are initiated. |
| timeoutSeconds | int | 1 | Number of seconds after which the probe times out. |
| periodSeconds | int | 10 | How often (in seconds) to perform the probe. |
| successThreshold | int | 1 | Minimum consecutive successes for the probe to be considered successful after having failed. |
| failureThreshold | int | 3 |  Minimum consecutive failures for the probe to be considered failed after having succeeded. |
| exec | array | {} |  Exec specifies the action to take. [More info](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.17/#execaction-v1-core) |
| httpGet | array | {} |  HTTPGet specifies the http request to perform. [More info](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.17/#httpgetaction-v1-core) |
| tcpSocket | array | {} |  TCPSocket specifies an action involving a TCP port. [More info](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.17/#tcpsocketaction-v1-core) |

## Custom Fluentd image

You can deploy custom images by overriding the default images using the following parameters in the fluentd or fluentbit sections of the logging resource.

| Name                    | Type           | Default | Description |
|-------------------------|----------------|---------|-------------|
| repository | string | "" | Image repository |
| tag | string | "" | Image tag |
| pullPolicy | string | "" | Always, IfNotPresent, Never |

The following example deploys a custom fluentd image:

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Logging
metadata:
  name: default-logging-simple
spec:
  fluentd:
    image:
      repository: banzaicloud/fluentd
      tag: v1.10.4-alpine-1
      pullPolicy: IfNotPresent
    configReloaderImage:
      repository: jimmidyson/configmap-reload
      tag: v0.4.0
      pullPolicy: IfNotPresent
    scaling:
      drain:
        image:
          repository: ghcr.io/banzaicloud/fluentd-drain-watch
          tag: v0.0.1
          pullPolicy: IfNotPresent
    bufferVolumeImage:
      repository: quay.io/prometheus/node-exporter
      tag: v1.1.2
      pullPolicy: IfNotPresent
  fluentbit: {}
  controlNamespace: logging
```

## KubernetesStorage

Define Kubernetes storage.

| Name      | Type | Default | Description |
|-----------|------|---------|-------------|
| hostPath | [HostPathVolumeSource](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#hostpathvolumesource-v1-core) | - | Represents a host path mapped into a pod. If path is empty, it will automatically be set to "/opt/logging-operator/<name of the logging CR>/<name of the volume>" |
| emptyDir | [EmptyDirVolumeSource](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#emptydirvolumesource-v1-core) | - | Represents an empty directory for a pod. |
| pvc | [PersistentVolumeClaim](#persistent-volume-claim) | - | A PersistentVolumeClaim (PVC) is a request for storage by a user. |

## Persistent Volume Claim

| Name      | Type | Default | Description |
|-----------|------|---------|-------------|
| spec | [PersistentVolumeClaimSpec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#persistentvolumeclaimspec-v1-core) | - | Spec defines the desired characteristics of a volume requested by a pod author. |
| source | [PersistentVolumeClaimVolumeSource](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#persistentvolumeclaimvolumesource-v1-core) | - | PersistentVolumeClaimVolumeSource references the user's PVC in the same namespace.  |

The Persistent Volume Claim should be created with the given `spec` and with the `name` defined in the `source`'s `claimName`.
