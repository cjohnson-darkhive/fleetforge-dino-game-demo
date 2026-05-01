# Demo Dino Deployment Example

This repository provides a minimal Kubernetes Deployment example for running and exposing an application within the FLEETFORGE platform.

It demonstrates:
- Deploying a containerized application
- Accessing platform-provided environment variables
- Pulling images from the approved private registry
- Exposing an application using host networking (edge-friendly pattern)

---

## Overview

The included manifest (`deployment.yaml`) deploys a simple application called `demo-dino`.

Key characteristics:
- Runs a single container
- Uses host networking for direct node-level access
- Automatically receives platform metadata via environment variables
- Pulls images from the approved private registry

---

## Prerequisites

Before applying this deployment, ensure:

- You have access to a Kubernetes cluster (k3s or equivalent)
- The cluster is configured to pull from the approved registry:
  - `images-private.darkhive.ai`
- The `regcred` secret exists in the target namespace

Example:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=images-private.darkhive.ai \
  --docker-username=<username> \
  --docker-password=<password>
```

---

## Deployment

Apply the manifest:

```bash
kubectl apply -f deployment.yaml
```

Verify the pod is running:

```bash
kubectl get pods
```

---

## Accessing the Application

This example uses:

```yaml
hostNetwork: true
```

This means the application is exposed directly on the node’s network.

Access it via:

```
http://<node-ip>:8085
```

No Service or Ingress is required for this pattern.

---

## Environment Variables

The deployment pulls environment variables from a shared ConfigMap:

```yaml
envFrom:
  - configMapRef:
      name: global-env
      optional: true
```

### What is `global-env`?

This ConfigMap is automatically populated by the platform and provides runtime context such as:

- Fleet name or ID
- Platform type
- System identifiers
- Other environment metadata

### Example usage inside your container

```bash
echo $FLEET_NAME
echo $PLATFORM_TYPE
echo $SYSTEM_ID
```

---

## Container Registry Requirements

Images must be pulled from the approved private registry:

```
images-private.darkhive.ai
```

- External registries such as Docker Hub or GHCR are not supported
- The `regcred` secret contains the credentials required to authenticate and pull images
- This secret must exist in the namespace before deploying

---

## Example Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-dino
  namespace: default
  labels:
    app: demo-dino
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-dino
  template:
    metadata:
      labels:
        app: demo-dino
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
        - name: demo-dino
          image: images-private.darkhive.ai/demo-dino:1.0.2
          ports:
            - containerPort: 8085
          envFrom:
            - configMapRef:
                name: global-env
                optional: true
      imagePullSecrets:
        - name: regcred
```

---

## Notes on hostNetwork

This deployment uses:

```yaml
hostNetwork: true
```

### Why

- Simplifies networking in edge environments
- Avoids needing Services or Ingress
- Works well for:
  - Field kits
  - Air gapped deployments
  - Rapid prototyping

### Tradeoffs

- Ports must be unique per node
- Less isolation than standard Kubernetes networking
- Not ideal for multi tenant clusters

---

## Next Steps

Once you are comfortable with this pattern, consider:

- Adding readiness and liveness probes
- Moving to a Service and Ingress model for scalable environments
- Defining a clear environment contract for your application
- Adding logging and metrics integration

---

## Troubleshooting

### Image pull errors
- Ensure `regcred` exists and is valid
- Confirm the image exists in `images-private.darkhive.ai`

### App not reachable
- Verify node IP
- Confirm container is listening on port 8085
- Check for port conflicts when using hostNetwork

### Missing environment variables
- Confirm `global-env` ConfigMap exists (optional but expected in platform environments)
