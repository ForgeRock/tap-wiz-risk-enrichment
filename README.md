# Wiz Risk Enrichment Node

A custom authentication node for PingAM and PingOne Advanced Identity Cloud (PingAIC) that enriches the authentication session with the user's Wiz risk profile — counting their open critical and high-severity issues and writing the resulting risk tier to shared state for downstream nodes and claims.

## Overview

During authentication, the node resolves the user's identity from shared state, authenticates to the Wiz API with client credentials, and queries the user's open issues (`OPEN` / `IN_PROGRESS`) with severity `CRITICAL` or `HIGH`. It writes the risk tier and issue counts to shared state and routes to `done`.

Use this node to:

- Attach a Wiz risk tier (`critical` / `high` / `none`) to the authentication session for risk-based routing
- Count the user's open critical/high issues for downstream decisions or claims
- Pair with the [Wiz Privileged Gate](https://github.com/ForgeRock/tap-wiz-privileged-gate) for privileged-access journeys, or use standalone for claims enrichment

### Risk tier logic

| Condition | Tier |
|---|---|
| At least one open `CRITICAL` issue | `critical` |
| No critical, at least one open `HIGH` issue | `high` |
| Neither | `none` |
| Error at any point | `unknown` (written on the error path — see error handling) |

## Compatibility

| Product | Versions |
|---|---|
| PingAM | 8.0.2 and later |
| PingOne Advanced Identity Cloud | All current tenants |

Built against the PingAM 8.0.2 node SDK (Java 17 bytecode).

## Outcomes

| Outcome | Description |
|---|---|
| `done` | Enrichment completed; shared state carries the risk tier and counts |
| `error` | The user id was missing, the Wiz configuration was incomplete, or the API call failed |

## Configuration

| Option | Type | Required | Default | Description |
|---|---|---|---|---|
| Auth URL | String | No | `https://auth.app.wiz.io/oauth/token` | Wiz OAuth 2.0 token endpoint. |
| API URL | String | Yes | `https://api.us17.app.wiz.io/graphql` | Tenant-specific Wiz GraphQL endpoint (varies by tenant region — copy from your Wiz tenant settings). |
| Client ID | String | Yes | *(empty)* | Wiz service-account client ID. |
| Client Secret | String | Yes | *(empty)* | Wiz service-account client secret. |

A fresh OAuth token (`grant_type=client_credentials`, `audience=wiz-api`) is obtained per node invocation; there is no token caching.

## Shared state

### Reads

| Key | Description |
|---|---|
| `mail` | User id used for the Wiz search — preferred |
| `username` | Fallback user id when `mail` is absent |

### Writes

| Key | Type | When | Description |
|---|---|---|---|
| `wizRiskTier` | String | Success, or any error (`unknown`) | `critical`, `high`, `none`, or `unknown` |
| `wizCriticalCount` | Number | Success | Open critical-severity issues |
| `wizHighCount` | Number | Success | Open high-severity issues |
| `wizCheckedAt` | String | Success | ISO-8601 UTC timestamp of the check |

**Note:** this node writes `wizRiskTier=unknown` on its error path — the only one of the four companion nodes that writes shared state on error. Downstream nodes can therefore distinguish "checked, no risk" (`none`) from "check failed" (`unknown`).

## Error handling

| Condition | Behavior |
|---|---|
| Neither `mail` nor `username` in shared state | `error` outcome, no outbound call |
| Blank auth/API URL, client ID, or secret | `error` outcome (validated at token-request time) |
| Token or GraphQL HTTP failure | `error` outcome |
| GraphQL response contains `errors` | `error` outcome |
| Any other exception | `error` outcome |

On every error path the node writes `wizRiskTier=unknown` and logs the cause. The `error` outcome never carries a failure-node message.

Fail open (continue to `Success`) or fail closed (route to `Failure`) is journey wiring, not node behavior.

## Journey example

```
Start → [Username/Password collection] → [Data Store Decision] → [Wiz Risk Enrichment]
                                                                    ├─ done  → [risk-based routing / claims]
                                                                    └─ error → Success or Failure (your policy)
```

To surface the tier in session claims, map `wizRiskTier` to a session property downstream of this node.

## Wiz service account

This node requires a Wiz service account with API access and the `read:issues` scope (or equivalent role allowing issue search). Create one in your Wiz tenant under **Settings → Service Accounts**, then copy the **API URL** (tenant-specific, e.g. `https://api.us17.app.wiz.io/graphql`) and the client credentials into the node configuration.

## Installation

1. Obtain the node jar (`wiz-gti-marketplace-nodes-<version>.jar` — this jar also contains the companion nodes; see Related nodes).
2. Copy it into `webapps/openam/WEB-INF/lib/` (PingAM) or deploy per your PingAIC packaging process.
3. Restart the AM web application container.
4. The node appears in the journey designer as **Wiz issues enrichment for the authenticated user**.

## Building from source

Requirements: JDK 17+, Maven 3.8+, and Ping Identity repository credentials (Backstage subscription) configured in `~/.m2/settings.xml` for `https://maven.forgerock.org/artifactory/private-releases`.

```bash
mvn clean package
```

The jar is produced in `target/`.

## Related nodes

- [Wiz Privileged Gate](https://github.com/ForgeRock/tap-wiz-privileged-gate)
- [Wiz Cloud Identity Check](https://github.com/ForgeRock/tap-wiz-cloud-identity-check)
- [GTI IP Reputation](https://github.com/ForgeRock/tap-gti-ip-reputation)

## Support

This integration is provided as a sample journey node. Review with your Ping Identity team before production use. For general PingAM/PingAIC node development, see the Ping Identity documentation on custom authentication nodes.
