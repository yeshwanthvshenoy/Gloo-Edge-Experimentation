# Gloo Edge as an API Gateway

This repository contains my experimentation with Gloo Edge as an API Gateway for Cloud Native Kubernetes services.

## Prerequisites

1. Docker Desktop
2. Kubectl
3. Glooctl
4. Helm
5. grpccurl

## What is covered in this?

- Run Gloo Edge as an API Gateway on kube cluster
- Run a gRPC based microservice on kube cluster
- Configure the gRPC service for endpoints and function definitions
- Run a HTTP based microservice on kube cluster
- Configure the service discovery to scan for Swagger docs
- Run a virtual service and use that to route requests to each service

## Steps for setup

```bash
# Install gloo edge as a Gateway using glooctl
$glooctl install gateway
```

### gRPC Setup

```bash
# Create gRPC deployment and service exposing on port 8001
$kubectl apply -f grpc.yaml

# Label and enabled function discovery for gRPC
$kubectl label upstream -n gloo-system default-grpc-petstore-8001 discovery.solo.io/function_discovery=enabled

# Check if gRPC Services are listed
$kubectl get upstream -n gloo-system default-grpc-petstore-8001 -o yaml
```

### HTTP Setup

```bash
# Create HTTP deployment and service exposing on port 8002
$kubectl apply -f http.yaml

# Check if HTTP Service is discovered and configured as upstream
$glooctl get upstream default-petstore-8002 --output kube-yaml

# Enabled swagger docs if any or any function discovery
$kubectl label namespace default  discovery.solo.io/function_discovery=enabled

# Check if swagger docs are found and populated the endpoints
$glooctl get upstream default-petstore-8002
```

## Apply Routing

```bash
# Create Virtual Service for HTTP and gRPC routes
$kubectl apply -f virtual-service.yaml
```

## Verification

### gRPC Routing

```bash
# Check if ServerReflection and StoreService are listed properly
$grpcurl -plaintext $(glooctl proxy address --port http) list

# Describe all functions in Store Service
$grpcurl -plaintext $(glooctl proxy address --port http) describe solo.examples.v1.StoreService

# List items in store
$grpcurl -plaintext $(glooctl proxy address --port http) solo.examples.v1.StoreService/ListItems

# Test create a new item
$grpcurl -plaintext -d '{"item":{"name":"item1"}}' $(glooctl proxy address --port http) solo.examples.v1.StoreService/CreateItem
```

### HTTP Routing

```bash
# List all pets
$curl $(glooctl proxy url)/all-pets
```

## Additional Experiments

### Prerequisites

1. OpenSSL

### Server TLS

```bash
# Generate a certificate using OpenSSL
$openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=petstore.example.com"

# Add the generated cert to kube secret
$kubectl create secret tls upstream-tls --key tls.key --cert tls.crt --namespace gloo-system

# Add the cert to our virtual service from kube secret
$glooctl edit virtualservice --name vs --namespace gloo-system --ssl-secret-name upstream-tls --ssl-secret-namespace gloo-system

# Verify if HTTP service is working, using -k since it is Self Signed Certificate
$curl -k $(glooctl proxy url --port https)/all-pets

# Verify if gRPC service is working
$grpcurl $(glooctl proxy address --port https) solo.examples.v1.StoreService/ListItems
```
