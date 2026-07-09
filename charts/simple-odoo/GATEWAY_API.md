# Gateway API Configuration

This chart supports both traditional Kubernetes Ingress and the newer Gateway API (nginx-gateway-fabric).

## Switching to Gateway API

To use Gateway API instead of Ingress, set `ingress.type` to `"gateway-api"`:

```yaml
ingress:
  enabled: true
  type: "gateway-api"  # Options: "ingress" or "gateway-api"
  domains:
    - url: "example.com"
  ports:
    - path: "/"
      name: "web"
      number: 8069
  gatewayApi:
    listenerName: "http"
    listenerPort: 80
    gatewayNamespace: "nginx-gateway"
    gatewayName: "nginx-gateway"
```

## Prerequisites

Before using Gateway API, ensure the nginx-gateway-fabric is installed in your cluster:

```bash
helm repo add nginx-stable https://helm.nginx.com/stable
helm install nginx-gateway nginx-stable/nginx-gateway \
  --namespace nginx-gateway \
  --create-namespace
```

## Gateway Configuration

By default, the chart assumes a Gateway resource already exists in your cluster. To have the chart create the Gateway resource, enable `createGateway`:

```yaml
ingress:
  type: "gateway-api"
  gatewayApi:
    createGateway: true
    gatewayName: "nginx-gateway"
    gatewayNamespace: "nginx-gateway"
    listenerName: "http"
    listenerPort: 80
```

## TLS/HTTPS with Gateway API

When `ingress.tls.enabled` is true, the chart automatically configures:
- An HTTPS listener on port 443
- Certificate references for all configured domains

Ensure your TLS certificates are created as Kubernetes secrets in the same namespace:

```bash
kubectl create secret tls domain-tls-secret \
  --cert=path/to/cert.crt \
  --key=path/to/key.key \
  -n <release-namespace>
```

## Migration from Ingress to Gateway API

1. Update your `values.yaml`:
   ```yaml
   ingress:
     type: "gateway-api"
     # Keep all domain and port configurations
   ```

2. If using a new Gateway, enable creation:
   ```yaml
   ingress:
     gatewayApi:
       createGateway: true
   ```

3. Deploy the updated chart:
   ```bash
   helm upgrade <release-name> . -f values.yaml
   ```

The old Ingress resources will be cleaned up automatically if using the default cleanup policy.

## HTTPRoute Details

When using Gateway API, the chart creates:
- **HTTPRoute**: Routes traffic based on hostname and path to your service
- **Gateway** (optional): Acts as the entry point for external traffic

HTTPRoute rules are created for each port/path combination in your configuration:

```yaml
rules:
  - matches:
      - path:
          type: PathPrefix
          value: "/"
    backendRefs:
      - name: service-name
        port: 8069
```

## Reference

- [Gateway API Documentation](https://gateway-api.sigs.k8s.io/)
- [nginx-gateway-fabric Documentation](https://docs.nginx.com/nginx-gateway-fabric/)
