---
name: Health check endpoint
description: Expose a lightweight GET /health endpoint reporting liveness, version and uptime.
---

# Health check endpoint

## Goal

Let load balancers, orchestrators and humans know whether the service is alive with a single cheap request.

## Requirements

### Functional

- `GET /health` answers `200` with a JSON body:
  ```json
  { "status": "ok", "version": "1.4.0", "uptimeSeconds": 1234 }
  ```
- `version` comes from the build or package metadata, never hard-coded.
- `uptimeSeconds` is measured from process start.

### Non-functional

- No authentication, no database or network call: liveness only.
- Responds in under 10 ms; never logged at info level to avoid noise.
- Responses are not cached (`Cache-Control: no-store`).

## Approach

Register one route in the existing HTTP layer, following the project's routing conventions. Capture the start instant once at boot and compute uptime per request.

## Steps

1. Read the application version from the project's build metadata.
2. Store the process start instant at boot.
3. Add the `GET /health` route returning the payload above with `Cache-Control: no-store`.
4. Exclude the route from authentication and request logging.
5. Document the endpoint where the project documents its API.

## Acceptance criteria

- `GET /health` returns `200` and the three fields with correct types.
- The endpoint works without credentials.
- Uptime increases between two calls.
