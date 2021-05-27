# Updating Astronomer's Helm Chart for Apache Airflow

This file documents any backwards-incompatible changes in the Astronomer Airflow Helm chart and
assists users migrating to a new version.

## 1.0.0

`1.0.0` takes a new approach by building on top of the official community Helm chart. This changes requires some renaming and adjusting in your values, documented below.

### Renamed Parameters

| From                                      | To                                        |
| ----------------------------------------- | ----------------------------------------- |
| `createUserJobAnnotations`                | `createUserJob.annotations`               |
| `runMigrationsJobAnnotations`             | `migrateDatabaseJob.annotations`          |
| `rbacEnabled`                             | `rbac.create`                             |
| `scheduler.airflowLocalSettings`          | `airflowLocalSettings`                    |
| `pgbouncer.extraIniDatabaseMetatdata` (yes, typo is correct) | `pgbouncer.extaIniMetadata` |
| `pgbouncer.extraIniDatabaseResultBackend` | `pgbouncer.extaIniResultBackend`          |
| `pgbouncer.extraIniPgbouncerConfig`       | `pgbouncer.extaIni`                       |

### ServiceAccount Annotations

ServiceAcccounts, and the annotations that are applied to them, are now separated for each component of Airflow.

For example, previously you would do:

```yaml
airflow:
  serviceAccountAnnotations:
    some-annotation: hello
```

Now you would do the following to apply the annotation to the scheduler ServiceAccount:

```yaml
airflow:
  scheduler:
    serviceAccount:
      annotations:
        some-annotation: hello
```

### Environment Variables from a Secret

Instead of using `mountAllFromSecretName`, you now use `extraEnvFrom`:

```yaml
airflow:
  extraEnvFrom: |-
    - secretRef:
      name: 'some-secret'
```

### Additional Volume for Workers

Instead of using `workers.additionalVolume`, you now use `workers.extraVolumes`, `workers.extraVolumeMounts`, and `extraObjects`:

```yaml
airflow:
  workers:
      extraVolumes:
         - name: worker-volume
           persistentVolumeClaim:
             claimName: worker-claim
      extraVolumeMounts:
        - name: worker-volume
          mountPath: /some-mount-path
extraObjects:
  - apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: {{ .Release.Name }}-worker-pv
      labels:
        tier: airflow
        component: worker
        release: {{ .Release.Name }}
        chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
        heritage: {{ .Release.Service }}
    {{- with .Values.airflow.labels }}
    {{ toYaml . | indent 4 }}
    {{- end }}
    spec:
      capacity:
        storage: 5Gi
      volumeMode: Filesystem
      accessModes:
        - ReadWriteMany
      persistentVolumeReclaimPolicy: Delete
  - apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: {{ .Release.Name }}-worker-claim
      labels:
        tier: airflow
        component: worker
        release: {{ .Release.Name }}
        chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
        heritage: {{ .Release.Service }}
    {{- with .Values.airflow.labels }}
    {{ toYaml . | indent 4 }}
    {{- end }}
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 5Gi
```