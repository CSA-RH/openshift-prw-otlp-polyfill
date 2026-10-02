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
          image: prom/prometheus:v2.47.0
          args:
            - "--config.file=/etc/prometheus/prometheus.yml"
            - "--storage.tsdb.path=/prometheus"
            - "--web.enable-otlp-receiver"
          ports:
            - containerPort: 9090
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
> `oc policy add-role-to-user \
>     system:image-puller \
>     system:serviceaccount:YOUR_TARGET_NAMESPACE:default \
>     --namespace=metrics-otlp)`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: metrics-otlp-bridge
---
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: telegraf-polyfill
  namespace: metrics-otlp-bridge
---
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: telegraf-polyfill-build
  namespace: metrics-otlp
spec:
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
      RUN microdnf install -y tar gzip && \
          curl -sL https://dl.influxdata.com/telegraf/releases/telegraf-${TELEGRAF_VERSION}_linux_amd64.tar.gz | tar xz --strip-components=2 -C /
      
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

