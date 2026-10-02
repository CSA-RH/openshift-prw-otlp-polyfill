# OpenShift PRW to OTLP Polyfill Bridge

A stateless, push-based translation layer designed to bridge the gap between OpenShift native Prometheus Remote Write 1.0 (PRW) and the OpenTelemetry (OTLP) standard. This repository provides a secure, enterprise-grade polyfill using Telegraf on top of Red Hat UBI 9 Micro, built entirely within the OpenShift cluster using BuildConfig.

This repository provides a secure, enterprise-grade polyfill using **Telegraf (MIT License)** on top of **Red Hat UBI 9 Micro**, allowing highly regulated environments to ingest cluster metrics into an OTel Collector without relying on application scraping.

## Prerequisites

Before starting, ensure the **Red Hat build of OpenTelemetry** operator is installed on your cluster. This operator is required to spin up the OpenTelemetry Collector instances.

## 1. Deploy the Standalone Prometheus OTLP Backend

To visualize the metrics, we will deploy a standalone Prometheus instance with native OTLP ingestion enabled.

> **Disclaimer**: *We are using a standalone Prometheus deployment instead of a `MonitoringStack` provided by the Cluster Observability Operator (COO). The COO is highly opinionated and currently does not allow injecting the arbitrary feature flags (such as `--web.enable-otlp-receiver`) required to accept direct OTLP pushes.*

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: metrics-otlp
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-otlp
  namespace: metrics-otlp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-otlp
  template:
    metadata:
      labels:
        app: prometheus-otlp
    spec:
      containers:
        - name: prometheus
          image: prom/prometheus:v3.13.2
          args:
            - "--config.file=/etc/prometheus/prometheus.yml"
            - "--storage.tsdb.path=/prometheus"
            - "--web.enable-otlp-receiver"
          ports:
            - containerPort: 9090          
          volumeMounts:
            - name: prometheus-storage
              mountPath: /prometheus      
      volumes:
        - name: prometheus-storage
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus-otlp
  namespace: metrics-otlp
spec:
  selector:
    app: prometheus-otlp
  ports:
    - port: 9090
      targetPort: 9090
```

## 2. Create the OpenTelemetry Collector

This Collector will receive the translated OTLP gRPC traffic from our Telegraf polyfill, inject the required Prometheus labels, and export it to the standalone Prometheus via OTLP HTTP.

```yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otel-poc
  namespace: metrics-otlp
spec:
  mode: deployment
  config:
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
    processors:
      transform:
        metric_statements:
          - context: resource
            statements:
              - set(attributes["job"], attributes["service.name"])
              - set(attributes["instance"], attributes["service.instance.id"])
    exporters:
      otlphttp/prometheus:
        endpoint: "http://prometheus-otlp.metrics-otlp.svc.cluster.local:9090/api/v1/otlp/v1/metrics"
        tls:
          insecure: true
      debug:
        verbosity: detailed
    service:
      pipelines:
        metrics:
          receivers: [otlp]
          processors: [transform]
          exporters: [debug, otlphttp/prometheus]
```

## 3. Build the Secure Polyfill Image via BuildConfig

Instead of relying on external registries, we use OpenShift native BuildConfig to assemble a multi-stage image. It downloads Telegraf using UBI Minimal and copies the binary into a UBI Micro image to achieve the smallest and most secure footprint.

> **Disclaimer regarding image location:** *The resulting image will be stored securely within the OpenShift internal registry. If you decide to deploy the polyfill in a different namespace, you must grant RBAC permissions to allow the default service account to pull from the image stream.*
> (Example:
> `oc policy add-role-to-user system:image-puller system:serviceaccount:YOUR_TARGET_NAMESPACE:default --namespace=metrics-otlp)`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: metrics-otlp-bridge
---
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: telegraf-polyfill-ubi
  namespace: metrics-otlp-bridge
---
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: telegraf-polyfill-build
  namespace: metrics-otlp-bridge
spec:
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
  output:
    to:
      kind: ImageStreamTag
      name: telegraf-polyfill-ubi:latest
  strategy:
    dockerStrategy: {}
  source:
    type: Dockerfile
    dockerfile: |
      FROM registry.access.redhat.com/ubi9/ubi-minimal:latest AS builder
      ARG TELEGRAF_VERSION=1.40.0
      # Detect architecture and extract with strip-components=1
      RUN microdnf install -y tar gzip && \
          ARCH=$(uname -m) && \
          if [ "$ARCH" = "aarch64" ]; then TG_ARCH="arm64"; else TG_ARCH="amd64"; fi && \
          curl -sL https://dl.influxdata.com/telegraf/releases/telegraf-${TELEGRAF_VERSION}_linux_${TG_ARCH}.tar.gz | tar xz --strip-components=1 -C /
      
      FROM registry.access.redhat.com/ubi9/ubi-micro:latest
      COPY --from=builder /usr/bin/telegraf /usr/bin/telegraf
      USER 1001
      EXPOSE 19291
      ENTRYPOINT ["/usr/bin/telegraf"]
```

Run this command to trigger the build:

```bash
oc start-build telegraf-polyfill-build -n metrics-otlp-bridge --follow
```

## 5. Configure OpenShift Monitoring and User Workload Monitoring

To enable Remote Write, you must update the Cluster Monitoring Operator's configuration maps.

> **Note:** *Cluster Monitoring generates a massive volume of metrics. Sending all of them via Remote Write without filtering can cause severe write latency, drain cluster resources, and delay the pipeline. It is highly recommended to use `writeRelabelConfigs` to drop high-cardinality or unnecessary metrics.*

### For Cluster Monitoring (Core OpenShift metrics):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
    prometheusK8s:
      remoteWrite:
        - url: "http://telegraf-translator-service.metrics-otlp-bridge.svc.cluster.local:19291/receive"
          writeRelabelConfigs:
            - sourceLabels: [__name__]
              regex: 'apiserver_request_duration_seconds_bucket|etcd_request_duration_seconds_bucket'
              action: drop
```

### For User Workload Monitoring (Application metrics):
Update or create the `user-workload-monitoring-config` ConfigMap in the `openshift-user-workload-monitoring` namespace.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-workload-monitoring-config
  namespace: openshift-user-workload-monitoring
data:
  config.yaml: |
    prometheus:
      remoteWrite:
        - url: "http://telegraf-translator-service.metrics-otlp-bridge.svc.cluster.local:19291/receive"
```

## Optional: Auditing PRW Traffic (Optional Sniffer)

To verify the exact PRW version emitted by OpenShift, you can deploy a lightweight netcat sniffer. This declarative setup includes a sed pipeline that safely drops the binary snappy/protobuf payload and only prints the clear-text HTTP headers to the pod logs.

### 6.1. Deploy the Sniffer

Apply the following Deployment and Service in the bridge namespace:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prw-sniffer
  namespace: metrics-otlp-bridge
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prw-sniffer
  template:
    metadata:
      labels:
        app: prw-sniffer
    spec:
      containers:
        - name: sniffer
          image: busybox
          command:
            - sh
            - -c
            - |
              while true; do
                nc -l -p 19291 | sed '/^\r*$/q'
                echo '-----------------------'
              done
          ports:
            - containerPort: 19291
---
apiVersion: v1
kind: Service
metadata:
  name: prw-sniffer-service
  namespace: metrics-otlp-bridge
spec:
  selector:
    app: prw-sniffer
  ports:
    - port: 19291
      targetPort: 19291
```

### 6.2. Redirect Traffic to the Sniffer

Temporarily update your OpenShift monitoring ConfigMap (as shown in Step 5) to point the `url` to the sniffer service instead of the Telegraf translator:

```yaml
remoteWrite:
  - url: "http://prw-sniffer-service.metrics-otlp-bridge.svc.cluster.local:19291/receive"
```

### 6.3. Read teh HTTP Readers

Tail the logs of the sniffer pod using its label to see the raw HTTP headers of the incoming metrics push:

```yaml
oc logs -l app=prw-sniffer -n metrics-otlp-bridge -f
```

Look for the `X-Prometheus-Remote-Write-Version` header in the output to determine if OCP is sending `0.1.0` or `2.0`.
```

Run this command to trigger the build:

```bash
oc start-build telegraf-polyfill-build -n metrics-otlp-bridge --follow
```

## 5. Configure OpenShift Monitoring and User Workload Monitoring

To enable Remote Write, you must update the Cluster Monitoring Operator's configuration maps.

> **Note:** *Cluster Monitoring generates a massive volume of metrics. Sending all of them via Remote Write without filtering can cause severe write latency, drain cluster resources, and delay the pipeline. It is highly recommended to use `writeRelabelConfigs` to drop high-cardinality or unnecessary metrics.*

### For Cluster Monitoring (Core OpenShift metrics):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
    prometheusK8s:
      remoteWrite:
        - url: "http://telegraf-translator-service.metrics-otlp-bridge.svc.cluster.local:19291/receive"
          writeRelabelConfigs:
            - sourceLabels: [__name__]
              regex: 'apiserver_request_duration_seconds_bucket|etcd_request_duration_seconds_bucket'
              action: drop
```

### For User Workload Monitoring (Application metrics):
Update or create the `user-workload-monitoring-config` ConfigMap in the `openshift-user-workload-monitoring` namespace.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-workload-monitoring-config
  namespace: openshift-user-workload-monitoring
data:
  config.yaml: |
    prometheus:
      remoteWrite:
        - url: "http://telegraf-translator-service.metrics-otlp-bridge.svc.cluster.local:19291/receive"
```

## Optional: Auditing PRW Traffic (Optional Sniffer)

To verify the exact PRW version emitted by OpenShift, you can deploy a lightweight netcat sniffer. This declarative setup includes a sed pipeline that safely drops the binary snappy/protobuf payload and only prints the clear-text HTTP headers to the pod logs.

### 6.1. Deploy the Sniffer

Apply the following Deployment and Service in the bridge namespace:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prw-sniffer
  namespace: metrics-otlp-bridge
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prw-sniffer
  template:
    metadata:
      labels:
        app: prw-sniffer
    spec:
      containers:
        - name: sniffer
          image: busybox
          command:
            - sh
            - -c
            - |
              while true; do
                nc -l -p 19291 | sed '/^\r*$/q'
                echo '-----------------------'
              done
          ports:
            - containerPort: 19291
---
apiVersion: v1
kind: Service
metadata:
  name: prw-sniffer-service
  namespace: metrics-otlp-bridge
spec:
  selector:
    app: prw-sniffer
  ports:
    - port: 19291
      targetPort: 19291
```

### 6.2. Redirect Traffic to the Sniffer

Temporarily update your OpenShift monitoring ConfigMap (as shown in Step 5) to point the `url` to the sniffer service instead of the Telegraf translator:

```yaml
remoteWrite:
  - url: "http://prw-sniffer-service.metrics-otlp-bridge.svc.cluster.local:19291/receive"
```

### 6.3. Read teh HTTP Readers

Tail the logs of the sniffer pod using its label to see the raw HTTP headers of the incoming metrics push:

```yaml
oc logs -l app=prw-sniffer -n metrics-otlp-bridge -f
```

Look for the `X-Prometheus-Remote-Write-Version` header in the output to determine if OCP is sending `0.1.0` or `2.0`.
