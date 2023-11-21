# Ever-Gauzy

[Ever-Gauzy](https://github.com/ever-co/ever-gauzy)  An Open Business Management Platform for Collaborative, On-Demand and Sharing Economies.

- Enterprise Resource Planning (ERP) software.
- Customer Relationship Management (CRM) software.
- Human Resource Management (HRM) software with employee Time and Activity Tracking functionality.
- Work and Project Management software.


## TL;DR;


Note that the default configuration is not suitable for production since it uses a  file database by default.
It is strongly recommended to use a dedicated database with Ever-gauzy.

For more examples see the [examples](examples) folder.

## Introduction

This chart bootstraps a [Ever-gauzy](https://github.com/ever-co/ever-gauzy) StatefulSet on a [Kubernetes](https://kubernetes.io) cluster using the [Helm](https://helm.sh) package manager.
It provisions a fully featured Ever Gauzy installation.
For more information on Ever-Gauzy and its capabilities, see its [documentation](https://github.com/ever-co/ever-gauzy).

## Installing the Chart

To install the chart with the release name `ever-gauzy`:

```console
$ helm install ever-gauzy everco/ever-gauzy
```

## Uninstalling the Chart

To uninstall the `ever-gauzy` deployment:

```console
$ helm uninstall ever-gauzy
```

## Configuration

The following table lists the configurable parameters of the Ever-gauzy chart and their default values.
---------------------------------
-----------------------------
Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`. For example:

```console
$ helm install ever-gauzy everco/ever-gauzy -n ever-gauzy --set replicas=1
```

Alternatively, a YAML file that specifies the values for the parameters can be provided while
installing the chart. For example:

```console
$ helm install ever-gauzy  everco/ever-gauzy -n ever-gauzy --values values.yaml
```

The chart offers great flexibility.
It can be configured to work with the official Ever-gauzy Docker image but any custom image can be used as well.

For the official Docker image, please check it's configuration at https://github.com/ever-co/ever-gauzy
### Usage of the `tpl` Function

The `tpl` function allows us to pass string values from `values.yaml` through the templating engine.
It is used for the following values:

* `extraInitContainers`
* `extraContainers`
* `extraEnv`
* `extraEnvFrom`
* `affinity`
* `extraVolumeMounts`
* `extraVolumes`
* `livenessProbe`
* `readinessProbe`
* `startupProbe`
* `topologySpreadConstraints`

Additionally, custom labels and annotations can be set on various resources the values of which being passed through `tpl` as well.

It is important that these values be configured as strings.
Otherwise, installation will fail.
See example for Google Cloud Proxy or default affinity configuration in `values.yaml`.



#### Using an External Database

The Keycloak Docker image supports various database types.
Configuration happens in a generic manner.

##### Using a Secret Managed by the Chart

The following examples uses a PostgreSQL database with a secret that is managed by the Helm chart.

```yaml
dbchecker:
  enabled: true

database:
  vendor: postgres
  hostname: mypostgres
  port: 5432
  username: '{{ .Values.dbUser }}'
  password: '{{ .Values.dbPassword }}'
  database: mydb
```

`dbUser` and `dbPassword` are custom values you'd then specify on the commandline using `--set-string`.

##### Using an Existing Secret

The following examples uses a PostgreSQL database with an existing secret.

```yaml
dbchecker:
  enabled: true

database:
  vendor: postgres
  hostname: mypostgres
  port: 5432
  database: mydb
  username: db-user
  existingSecret: byo-db-creds # Password is retrieved via .password
```



### High Availability and Clustering

For high availability, Ever-gauzy must be run with multiple replicas (`replicas > 1`).
The chart has a helper template (`ever-gauzy.serviceDnsName`) that creates the DNS name based on the headless service.



#### Custom Service Discovery

If a custom JGroups discovery is needed, then you can configure:

```yaml
cache:
  stack: custom
```



#### Autoscaling

Due to the caches in Keycloak only replicating to a few nodes (two in the example configuration above) and the limited controls around autoscaling built into Kubernetes, it has historically been problematic to autoscale Keycloak.
However, in Kubernetes 1.18 [additional controls were introduced](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#support-for-configurable-scaling-behavior) which make it possible to scale down in a more controlled manner.

The example autoscaling configuration in the values file scales from three up to a maximum of ten Pods using CPU utilization as the metric. Scaling up is done as quickly as required but scaling down is done at a maximum rate of one Pod per five minutes.

Autoscaling can be enabled as follows:

```yaml
autoscaling:
  enabled: true
```

KUBE_PING service discovery seems to be the most reliable mechanism to use when enabling autoscaling, due to being faster than DNS_PING at detecting changes in the cluster.

### Running Ever-Gauzy Behind a Reverse Proxy


### Using Google Cloud SQL Proxy

Depending on your environment you may need a local proxy to connect to the database.
This is, e. g., the case for Google Kubernetes Engine when using Google Cloud SQL.
Create the secret for the credentials as documented [here](https://cloud.google.com/sql/docs/postgres/connect-kubernetes-engine) and configure the proxy as a sidecar.

Because `extraContainers` is a string that is passed through the `tpl` function, it is possible to create custom values and use them in the string.

```yaml
database:
  vendor: postgres
  hostname: '127.0.0.1'
  port: 5432
  database: postgres
  username: myuser
  password: mypassword

# Custom values for Google Cloud SQL
cloudsql:
  project: my-project
  region: europe-west1
  instance: my-instance

extraContainers: |
  - name: cloudsql-proxy
    image: gcr.io/cloudsql-docker/gce-proxy:1.17
    command:
      - /cloud_sql_proxy
    args:
      - -instances={{ .Values.cloudsql.project }}:{{ .Values.cloudsql.region }}:{{ .Values.cloudsql.instance }}=tcp:5432
      - -credential_file=/secrets/cloudsql/credentials.json
    volumeMounts:
      - name: cloudsql-creds
        mountPath: /secrets/cloudsql
        readOnly: true

extraVolumes: |
  - name: cloudsql-creds
    secret:
      secretName: cloudsql-instance-credentials
```

### Changing the Context Path

By default, Ever-gauzy is served under context `/auth`.
Trailing slash is removed from path. This can be changed to another context path like `/` as follows:

```yaml
http:
  relativePath: '/'
```

Alternatively, you may supply it via CLI flag:

```console
--set-string http.relativePath=/
```

### Prometheus Metrics Support

#### Ever-gauzy Metrics

Ever-gauzy  can expose metrics via `/auth/metrics`.

Metrics are enabled by default via:
```yaml
metrics:
  enabled: true
```

Add a ServiceMonitor if using prometheus-operator:

```yaml
serviceMonitor:
  # If `true`, a ServiceMonitor resource for the prometheus-operator is created
  enabled: true
```

Checkout `values.yaml` for customizing the ServiceMonitor and for adding custom Prometheus rules.

Add annotations if you don't use prometheus-operator:

```yaml
service:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
```

#### Ever-gauzy  Metrics SPI



A separate `ServiceMonitor` can be enabled to scrape metrics from the SPI:

```yaml
extraServiceMonitor:
  # If `true`, an additional ServiceMonitor resource for the prometheus-operator is created
  enabled: true
```

Checkout `values.yaml` for customizing this ServiceMonitor.

Note that the metrics endpoint is exposed on the HTTP port.
You may want to restrict access to it in your ingress controller configuration.
For ingress-nginx, this could be done as follows:

```yaml
annotations:
  nginx.ingress.kubernetes.io/server-snippet: |
    location ~* /auth/realms/[^/]+/metrics {
        return 403;
    }
```

## Why StatefulSet?

The headless service that governs the StatefulSet is used for DNS discovery via DNS_PING.

## Bad Gateway and Proxy Buffer Size in Nginx

A common issue with Ever-Gauzy and nginx is that the proxy buffer may be too small for what Ever-gauzy is trying to send. This will result in a Bad Gateway (502) error. There are [many](https://github.com/kubernetes/ingress-nginx/issues/4637) [issues](https://stackoverflow.com/questions/56126864/why-do-i-get-502-when-trying-to-authenticate) around the internet about this. The solution is to increase the buffer size of nginx. This can be done by creating an annotation in the ingress specification:

```yaml
ingress:
  annotations:
    nginx.ingress.kubernetes.io/proxy-buffer-size: "128k"
```

## Upgrading

Notes for upgrading from previous Ever-gauzy chart versions.

### From chart < 18.0.0

* Ever-gauzy is updated to 1.0.1
* Added new `health.enabled` option.

Ever-gauzy 1.0.1 allows to enable the health endpoint independently of the metrics endpoint via the `health-enabled` setting. 
We reflect that via the new config option `health.enabled`.

Please read the additional notes about [Migrating to 1.0.1](https://github.com/ever-co/ever-gauzy) in the Ever-gauzy documentation.