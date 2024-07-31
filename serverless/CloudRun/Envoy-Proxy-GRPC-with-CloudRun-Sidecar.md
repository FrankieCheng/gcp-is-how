# Sidecar Envoy with grpc on Cloud Run

# Overview

Cloud Run multi-container is released. In this article, we will use the multi-container to build a gRPC-Web service with an Envoy sidecar in a Cloud Run service.

The first half of this article is about deploying the service to Cloud Run and calling it from the web.

# Environment Setup

This assumes a new project and operation in Cloud Shell. If you are operating locally, please install the necessary tools as appropriate.

Download the code from nownable demo as below.

```
git clone https://github.com/nownabe/google-cloud-examples
```

# Solution

## Overview

[https://cloud.google.com/blog/products/serverless/cloud-run-now-supports-multi-container-deployments](https://cloud.google.com/blog/products/serverless/cloud-run-now-supports-multi-container-deployments)  
![Cloud Run with Envoy](https://storage.googleapis.com/gweb-cloudblog-publish/images/1_Cloud_Run.max-2000x2000.png)

Serving Container: Host Envoy with 8080
Sidecar Container: Host grpc service

## gRPC service

The backend gRPC service is implemented in Go, and you can access the service from the following link.  
[https://github.com/nownabe/google-cloud-examples/tree/main/grpc-web-envoy/go](https://github.com/nownabe/google-cloud-examples/tree/main/grpc-web-envoy/go)

## Envoy configuration

[https://github.com/nownabe/google-cloud-examples/blob/main/grpc-web-envoy/envoy/envoy.yaml](https://github.com/nownabe/google-cloud-examples/blob/main/grpc-web-envoy/envoy/envoy.yaml)

```
static_resources:
  listeners:
    - name: listener_0
      address:
        socket_address: {address: 0.0.0.0, port_value: 8080}
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                codec_type: AUTO
                stat_prefix: ingress_http
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: local_service
                      domains: ["*"]
                      routes:
                        - match: {prefix: "/"}
                          route:
                            cluster: grpc
                            timeout: 0s
                            max_stream_duration:
                              grpc_timeout_header_max: 0s
                      typed_per_filter_config:
                        envoy.filters.http.cors:
                          "@type": type.googleapis.com/envoy.extensions.filters.http.cors.v3.CorsPolicy
                          allow_origin_string_match:
                            - prefix: "*"
                          allow_methods: GET, PUT, DELETE, POST, OPTIONS
                          allow_headers: keep-alive,user-agent,cache-control,content-type,content-transfer-encoding,custom-header-1,x-accept-content-transfer-encoding,x-accept-response-streaming,x-user-agent,x-grpc-web,grpc-timeout
                          max_age: "1728000"
                          expose_headers: custom-header-1,grpc-status,grpc-message
                http_filters:
                  - name: envoy.filters.http.grpc_web
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.grpc_web.v3.GrpcWeb
                  - name: envoy.filters.http.cors
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.cors.v3.Cors
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
  clusters:
    - name: grpc
      type: STATIC
      lb_policy: ROUND_ROBIN
      typed_extension_protocol_options:
        envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
          "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
          explicit_http_config:
            http2_protocol_options: {}
      load_assignment:
        cluster_name: grpc
        endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address: {address: 127.0.0.1, port_value: 50051}

```

## Cloud Run service Configuration template with two containers.

[https://github.com/nownabe/google-cloud-examples/blob/main/grpc-web-envoy/service.template.yaml](https://github.com/nownabe/google-cloud-examples/blob/main/grpc-web-envoy/service.template.yaml)

```py
# See the reference
# https://cloud.google.com/run/docs/reference/yaml/v1
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: grpc-web
  labels:
    cloud.googleapis.com/location: us-central1
  annotations:
    run.googleapis.com/launch-stage: BETA
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "1"
        autoscaling.knative.dev/maxScale: "1"
        run.googleapis.com/execution-environment: gen1
    spec:
      serviceAccountName: ${SERVICE_ACCOUNT}
      containers:
        - image: ${ENVOY_IMAGE}
          name: envoy
          ports:
            - containerPort: 8080
          resources:
            limits:
              cpu: ".08"
              memory: 128Mi
        - image: ${GRPC_IMAGE}
          name: grpc
          resources:
            limits:
              cpu: ".08"
              memory: 128Mi
          livenessProbe:
            grpc:
              port: 50051
              service: bookstore.BookstoreService
```

# Steps

## Enable the GCP Services

Cloud Run  
Cloud Build  
Artifact Registry

```
gcloud services enable \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  run.googleapis.com
```

## Create Docker repository

```
gcloud artifacts repositories create \
  grpc-web \
  --location us-central1 \
  --repository-format DOCKER
```

## Create Service Account

```
gcloud iam service-accounts create grpc-web

gcloud projects add-iam-policy-binding \
  "$(gcloud config get project)" \
  --member "serviceAccount:$(gcloud projects describe $(gcloud config get project) --format 'value(projectNumber)')@cloudbuild.gserviceaccount.com" \
  --role "roles/run.admin"

gcloud iam service-accounts add-iam-policy-binding \
  "grpc-web@$(gcloud config get project).iam.gserviceaccount.com" \
  --member "serviceAccount:$(gcloud projects describe $(gcloud config get project) --format 'value(projectNumber)')@cloudbuild.gserviceaccount.com" \
  --role "roles/iam.serviceAccountUser"
```

## Build the images

Use Cloud Build to do the following tasks:

* Build and push the Docker image for the gRPC service (Go app)  
* Build and push the Docker image for Envoy  
* Draw the Cloud Run service definition file  
* Deploy the Cloud Run service  
* cloudbuilld.yaml is provided, so you only need to run one command.

```
cd google-cloud-examples/grpc-web-envoy
gcloud builds submit .
```

# Run the service

```
gcloud run services add-iam-policy-binding \
  grpc-web \
  --region us-central1 \
  --member "allUsers" \
  --role "roles/run.invoker"
```

Get the service url.

```
gcloud run services describe grpc-web --region us-central1 --format 'value(status.url)'

```

# Test Service

You can use the following web to test the application.  
[https://github.com/nownabe/google-cloud-examples/tree/main/grpc-web-envoy/web](https://github.com/nownabe/google-cloud-examples/tree/main/grpc-web-envoy/web)  