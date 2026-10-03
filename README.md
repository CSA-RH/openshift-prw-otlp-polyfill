# OpenShift PRW to OTLP Polyfill Bridge

A stateless, push-based translation layer designed to bridge the gap between OpenShift native Prometheus Remote Write 1.0 (PRW) and the OpenTelemetry (OTLP) standard. This repository provides a secure, enterprise-grade polyfill using Telegraf on top of Red Hat UBI 9 Micro, built entirely within the OpenShift cluster using BuildConfig.

This repository provides a secure, enterprise-grade polyfill using **Telegraf (MIT License)** on top of **Red Hat UBI 9 Micro**, allowing highly regulated environments to ingest cluster metrics into an OTel Collector without relying on application scraping.

## Prerequisites

Before starting, ensure the **Red Hat build of OpenTelemetry** operator is installed on your cluster. This operator is required to spin up the OpenTelemetry Collector instances.

## 1. Deploy the Standalone VictoriaMetrics OTLP Backend

We will deploy a single-node VictoriaMetrics instance to visualize the metrics. VictoriaMetrics serves as a drop-in replacement for Prometheus but natively accepts out-of-order samples and has OTLP ingestion enabled by default on its main port (8428), making it highly resilient for high-throughput environments.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: metrics-otlp
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: victoria-metrics
  namespace: metrics-otlp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: victoria-metrics
  template:
    metadata:
      labels:
        app: victoria-metrics
    spec:
      containers:
        - name: victoriametrics
          image: victoriametrics/victoria-metrics:v1.101.0
          args:
            # Data retention and storage path
            - "-retentionPeriod=15d"
            - "-storageDataPath=/storage"
            # Enable the web UI and OTLP ingestion on the same port
            - "-httpListenAddr=:8428"
          ports:
            - containerPort: 8428
          volumeMounts:
            - name: vm-storage
              mountPath: /storage
      volumes:
        # Using emptyDir for PoC purposes. Use a PVC for production workloads.
        - name: vm-storage
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: victoria-metrics
  namespace: metrics-otlp
spec:
  selector:
    app: victoria-metrics
  ports:
    - port: 8428
      targetPort: 8428
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: victoria-metrics-ui
  namespace: metrics-otlp
spec:
  to:
    kind: Service
    name: victoria-metrics
  port:
    targetPort: 8428
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

## 2. Create the OpenTelemetry Collector

This Collector receives the translated OTLP gRPC traffic from the Telegraf polyfill and exports it to VictoriaMetrics. We use a batch processor to handle the massive OpenShift throughput smoothly and ensure stable memory consumption.

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
            # NOTE: Expand receiver limit to 32 MB instead of 4. 
            max_recv_msg_size_mib: 32
            
    processors:
      # Inject required labels to maintain context from the original metrics
      transform:
        metric_statements:
          # 1. Replace the default Telegraf prefix with our custom PoC identifier
          - context: metric
            statements:              
              - replace_pattern(name, "^prometheus_remote_write_(.*)", "poc_prw_$$1")
          - context: resource
            statements:
              - set(attributes["job"], attributes["service.name"])
              - set(attributes["instance"], attributes["service.instance.id"])
      # Mandatory for high-throughput environments to prevent memory exhaustion
      batch:
        send_batch_size: 10000
        timeout: 1s
        
    exporters:
      # VictoriaMetrics natively supports OTLP via HTTP. 
      # The collector automatically appends '/v1/metrics' to the endpoint,
      # resulting in the exact native VM path: '/opentelemetry/v1/metrics'.
      otlphttp/victoriametrics:
        endpoint: "http://victoria-metrics.metrics-otlp.svc.cluster.local:8428/opentelemetry"
        tls:
          insecure: true
          
    service:
      pipelines:
        metrics:
          receivers: [otlp]
          processors: [transform, batch]
          exporters: [otlphttp/victoriametrics]
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
spec:
  lookupPolicy:
    local: true
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

## 4. Deploy the Telegraf Polyfill

Apply the ConfigMap, Deployment, and Service in the bridge namespace. The Telegraf container will run using the multi-stage image generated in the previous step, fetching it directly from the OpenShift internal registry.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: telegraf-translator-config
  namespace: metrics-otlp-bridge
data:
  telegraf.conf: |
    [agent]
      interval = "5s"
      flush_interval = "5s"
      omit_hostname = true
      metric_buffer_limit = 1000000
      metric_batch_size = 20000
      
    [[inputs.http_listener_v2]]
      service_address = ":19291"
      paths = ["/receive"]
      data_format = "prometheusremotewrite"
      
    [[outputs.opentelemetry]]
      service_address = "otel-poc-collector.metrics-otlp.svc.cluster.local:4317"
      timeout = "5s"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: telegraf-translator
  namespace: metrics-otlp-bridge
spec:
  replicas: 1
  selector:
    matchLabels:
      app: telegraf-translator
  template:
    metadata:
      labels:
        app: telegraf-translator
    spec:
      containers:
        - name: telegraf
          # Pulling the image from the internal registry within the bridge namespace
          image: telegraf-polyfill-ubi:latest
          ports:
            - containerPort: 19291
          volumeMounts:
            - name: config
              mountPath: /etc/telegraf
      volumes:
        - name: config
          configMap:
            name: telegraf-translator-config
---
apiVersion: v1
kind: Service
metadata:
  name: telegraf-translator-service
  namespace: metrics-otlp-bridge
spec:
  selector:
    app: telegraf-translator
  ports:
    - port: 19291
      targetPort: 19291
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

## 6. (Optional) Auditing PRW Traffic (Sniffer)

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
