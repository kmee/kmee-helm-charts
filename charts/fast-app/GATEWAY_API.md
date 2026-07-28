# Gateway API Configuration

Este chart suporta tanto o Ingress tradicional do Kubernetes quanto a Gateway API
(nginx-gateway-fabric).

## Alternando para Gateway API

Defina `ingress.type` como `"gateway-api"`:

```yaml
ingress:
  enabled: true
  type: "gateway-api"   # "ingress" ou "gateway-api"
  hosts:
    - host: barcode.local
      paths:
        - path: /
          pathType: Prefix
  gatewayApi:
    createGateway: false
    gatewayClassName: nginx
    listenerName: "http"
    listenerPort: 80
    gatewayNamespace: "nginx-gateway"
    gatewayName: "nginx-gateway"
```

Com `type: "gateway-api"` o template `ingress.yaml` não é renderizado; no lugar dele
sai um `HTTPRoute` (`gateway.networking.k8s.io/v1`) apontando para o Service do chart.

## Pré-requisitos

nginx-gateway-fabric (e as CRDs da Gateway API) instalados no cluster:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml
helm install ngf oci://ghcr.io/nginxinc/charts/nginx-gateway-fabric \
  --create-namespace -n nginx-gateway
```

## Mapeamento de valores

| Ingress                       | Gateway API                                   |
|-------------------------------|-----------------------------------------------|
| `ingress.hosts[].host`        | `spec.hostnames[]` do HTTPRoute               |
| `ingress.hosts[].paths[].path`| `spec.rules[].matches[].path.value`           |
| `pathType: Prefix`            | `type: PathPrefix`                            |
| `pathType: Exact`             | `type: Exact`                                 |
| `ingress.className`           | `ingress.gatewayApi.gatewayClassName`         |
| `ingress.annotations`         | `ingress.gatewayApi.annotations` (no HTTPRoute) |

O HTTPRoute é único e cobre todos os hostnames, então paths repetidos entre hosts
são deduplicados em uma só rule.

As anotações `nginx.ingress.kubernetes.io/*` **não** valem para Gateway API. Migração:

| Anotação do Ingress                          | Equivalente em Gateway API                            |
|----------------------------------------------|-------------------------------------------------------|
| `proxy-read-timeout` / `proxy-send-timeout`  | `ingress.gatewayApi.timeouts.{request,backendRequest}`|
| `proxy-body-size`                            | `ClientSettingsPolicy.body.maxSize` (NGF)             |
| `whitelist-source-range`                     | `SnippetsFilter` (`allow`/`deny`) + `extraFilters`    |
| `cert-manager.io/cluster-issuer`             | `ingress.gatewayApi.gatewayAnnotations` (gateway-shim)|
| `ssl-redirect`                               | `RequestRedirect` filter em `extraFilters`            |
| `proxy-connect-timeout`                      | sem equivalente direto no NGF                         |

`ClientSettingsPolicy`/`SnippetsFilter` são CRDs do NGF — aplique como manifests
próprios no namespace do release e referencie via `extraFilters`:

```yaml
ingress:
  gatewayApi:
    extraFilters:
      - type: ExtensionRef
        extensionRef:
          group: gateway.nginx.org
          kind: SnippetsFilter
          name: allowlist
```

## Gateway compartilhado vs. próprio

Padrão (`createGateway: false`): o chart só cria o HTTPRoute e anexa a um Gateway já
existente — recomendado quando várias apps compartilham o mesmo Gateway.

Com `createGateway: true` o chart também cria o Gateway em
`gatewayNamespace`/`gatewayName`. Não habilite em mais de um release apontando para o
mesmo nome, senão os releases disputam o mesmo recurso.

## TLS

Preencha `ingress.tls` normalmente. Quando `createGateway: true`, cada `secretName`
entra em `certificateRefs` de um listener HTTPS na porta 443:

```yaml
ingress:
  enabled: true
  type: "gateway-api"
  tls:
    - secretName: barcode-tls
      hosts:
        - barcode.local
  gatewayApi:
    createGateway: true
```

Por padrão os `certificateRefs` saem **sem** `namespace`, ou seja, o Secret é buscado
no namespace do Gateway — não precisa de `ReferenceGrant`. Se o Secret estiver em
outro namespace, informe `gatewayApi.certificateNamespace` e crie o `ReferenceGrant`
correspondente.

Com `gatewayAnnotations: {cert-manager.io/cluster-issuer: ...}` o cert-manager
(gateway-shim) emite o certificado a partir do listener e grava o Secret no namespace
do Gateway. Nesse caso deixe `certificateNamespace` vazio.

Atenção ao `parentRefs`: o `sectionName` precisa apontar para o listener https, senão
a rota só atende HTTP. Use `listenerNames: [http, https]` quando houver TLS.

## Verificação

```bash
kubectl get httproute
kubectl describe httproute <release>-fastapi-app
kubectl get gateway -n nginx-gateway
```

`status.parents[].conditions` do HTTPRoute com `Accepted=True` e `ResolvedRefs=True`
indica que o Gateway aceitou a rota.
