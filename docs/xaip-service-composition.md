# XAIP: Service Composition Schema

**Atom type:** `service`
**Version:** 0.1
**Audience:** Service engineers, API architects, platform integrators

---

## 1. Purpose

A service catalog composition assembles a service identity, protocol atoms, auth-scheme atoms, and endpoint-pattern atoms into a complete service definition. The composition is the canonical descriptor from which clients can be generated, OpenAPI specifications exported, and Universal Bus event routing configured — without requiring the service's source code to be read or the service to be running.

---

## 2. Composition Structure

A service composition is a JSON document at the path `atoms/<slug>/v<version>/atom.json` within the service-atoms catalog.

```json
{
  "atom_type": "service",
  "service_id": "convergent-systems-co/olympus-inference-api",
  "version": "1",
  "display_name": "Olympus Inference API",
  "service_identity_ref": "https://identity-atoms.convergent-systems.co/atoms/olympus-inference-svc/v1/atom.json",
  "protocols": [
    {
      "protocol_ref": "protocol:https-rest",
      "base_url": "https://inference.olympus.convergent-systems.co",
      "tls_required": true
    }
  ],
  "auth_scheme_refs": [
    "auth-method:oidc-pkce",
    "auth-method:api-key-hmac"
  ],
  "endpoint_patterns": [
    {
      "endpoint_id": "infer",
      "method": "POST",
      "path_pattern": "/v{version}/infer",
      "path_params": [
        { "name": "version", "type": "integer", "description": "API version" }
      ],
      "request_schema_ref": "schemas/infer-request.json",
      "response_schema_ref": "schemas/infer-response.json",
      "auth_required": true,
      "idempotent": false,
      "tags": ["inference", "core"]
    },
    {
      "endpoint_id": "health",
      "method": "GET",
      "path_pattern": "/health",
      "request_schema_ref": null,
      "response_schema_ref": "schemas/health-response.json",
      "auth_required": false,
      "idempotent": true,
      "tags": ["ops"]
    }
  ],
  "bus_registration": {
    "topics_published": ["inference.completed", "inference.failed"],
    "topics_subscribed": ["config.updated"]
  },
  "metadata": {
    "catalog_ref": "service-atoms.convergent-systems.co",
    "owner": "platform-team"
  }
}
```

### 2.1 Required fields

| Field | Type | Description |
|---|---|---|
| `service_id` | string | Stable, namespaced identifier |
| `version` | string | Service definition version (incremented on breaking change) |
| `service_identity_ref` | URI | The identity atom from identity-atoms that represents this service's registration |
| `protocols` | array | One or more transport protocol atoms with base URL |
| `auth_scheme_refs` | array | Auth-method atoms accepted for authenticated endpoints |
| `endpoint_patterns` | array | Endpoint descriptors (see §2.2) |

### 2.2 Endpoint pattern fields

| Field | Type | Required | Description |
|---|---|---|---|
| `endpoint_id` | string | yes | Stable identifier used in generated client code |
| `method` | string | yes | HTTP method (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`) |
| `path_pattern` | string | yes | Path with `{param}` placeholders |
| `request_schema_ref` | URI or null | no | JSON Schema for the request body |
| `response_schema_ref` | URI | yes | JSON Schema for the success response body |
| `auth_required` | boolean | yes | Whether a valid auth token must be presented |
| `idempotent` | boolean | yes | Whether repeated calls with the same input are safe |
| `tags` | string[] | no | Grouping labels for generated client organization |

---

## 3. OpenAPI Export Generation

The service composition maps directly to an OpenAPI 3.0 document. The export is deterministic: the same composition always produces the same OpenAPI document.

### 3.1 Mapping table

| Composition field | OpenAPI 3.0 field |
|---|---|
| `service_id` | `info.title` (slugified) |
| `version` | `info.version` |
| `protocols[*].base_url` | `servers[*].url` |
| `auth_scheme_refs` | `components.securitySchemes` (one entry per auth-method atom) |
| `endpoint_patterns[*].path_pattern` | `paths` keys (braces format preserved) |
| `endpoint_patterns[*].method` | `paths.<path>.<method>` |
| `endpoint_patterns[*].request_schema_ref` | `paths.<path>.<method>.requestBody.content.application/json.schema.$ref` |
| `endpoint_patterns[*].response_schema_ref` | `paths.<path>.<method>.responses.200.content.application/json.schema.$ref` |
| `endpoint_patterns[*].auth_required` | `paths.<path>.<method>.security` (empty array if false) |
| `endpoint_patterns[*].tags` | `paths.<path>.<method>.tags` |

### 3.2 Auth scheme mapping

Each entry in `auth_scheme_refs` resolves to an auth-method atom and maps to an OpenAPI `securityScheme`:

| Auth-method atom | OpenAPI securityScheme type |
|---|---|
| `auth-method:oidc-pkce` | `type: oauth2, flows.authorizationCode` with PKCE |
| `auth-method:api-key-hmac` | `type: apiKey, in: header, name: X-API-Key` |
| `auth-method:mtls` | `type: mutualTLS` |
| `auth-method:bearer-jwt` | `type: http, scheme: bearer, bearerFormat: JWT` |

### 3.3 Export command (CLI)

```bash
# Export OpenAPI 3.0 from a composed service atom
atoms export openapi \
  --service convergent-systems-co/olympus-inference-api \
  --version 1 \
  --output ./openapi/olympus-inference-api.yaml
```

---

## 4. Universal Bus Discovery and Event Routing

A composed service declares its event topology in `bus_registration`. This declaration is the input used by the Universal Bus to configure routing — no runtime service interrogation is required.

### 4.1 Bus registration fields

| Field | Type | Description |
|---|---|---|
| `topics_published` | string[] | Event topics this service emits. Topic names follow `<domain>.<event>` convention. |
| `topics_subscribed` | string[] | Event topics this service consumes. |
| `consumer_group` | string | Optional. Consumer group identifier for competing-consumer delivery on subscribed topics. |
| `dlq_topic` | string | Optional. Dead-letter topic for messages this service fails to process. |

### 4.2 Registration flow

On startup, the service reads its own composition atom and posts its `bus_registration` block to the Universal Bus discovery endpoint:

```
POST /bus/v1/register
{
  "service_ref": "https://service-atoms.convergent-systems.co/atoms/olympus-inference-api/v1/atom.json",
  "bus_registration": { ... }
}
```

The bus stores the registration and begins routing `config.updated` events to the service immediately. The service does not configure routing logic; the composition atom is the routing configuration.

### 4.3 Event schema conformance

Topics declared in `topics_published` MUST have a corresponding event-type atom in the event-atoms catalog:

```
event-type: inference.completed
  payload_schema_ref: schemas/inference-completed-event.json
  source_service_ref: service:convergent-systems-co/olympus-inference-api/v1
```

The bus validates published events against the event-type atom's `payload_schema_ref` at ingest.

---

## 5. Service Identity Integration

The `service_identity_ref` links the service to an identity atom from the identity-atoms catalog. This identity atom carries the service's auth-method configuration, trust level, and federation policy — the same structure as human identities, applied to machine principals.

### 5.1 Service identity atom example

```json
{
  "atom_type": "identity",
  "atom_id": "olympus-inference-svc",
  "version": "1",
  "auth_methods": [
    {
      "auth_method_ref": "auth-method:mtls",
      "primary": true
    }
  ],
  "trust_framework_ref": "trust-framework:olympus-service-mesh",
  "federation_policy": "strict-local",
  "trust_level": "signed"
}
```

The service presents this identity to peer services, to the Universal Bus, and to policy engines. Policy decisions for service-to-service calls use the `service_identity_ref` as the subject, evaluated against the same policy composition stack used for human identities.

---

## 6. Composition Conventions

| Convention | Value |
|---|---|
| Service atom path | `atoms/<slug>/v<version>/atom.json` |
| Protocol atom path | `atoms/protocols/<slug>/v<version>/atom.json` |
| Auth-method atom path | resolved from identity-atoms catalog |
| Endpoint schema path | `schemas/<endpoint-id>-<direction>.json` (co-located with atom) |
| OpenAPI export path | `openapi/<service-id>.yaml` (generated; not committed to source) |

---

## 7. Related Atoms and Docs

- `protocol` atom — transport layer (HTTPS REST, gRPC, WebSocket, SSE)
- `endpoint-pattern` atom — method, path, and schema contract for one endpoint
- identity-atoms: `xaip-identity-composition.md` — service identity registration and trust
- event-atoms: event-type atoms — payload schemas for bus-published events
- policy-atoms: `xaip-policy-composition.md` — policy stacks applied to service-to-service calls
- plugin-atoms: `xaip-plugin-convention.md` — service-backed plugin trust and permission scope
