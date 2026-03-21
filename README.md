# Dog Rescue Adoption API

An [OpenAPI 3.0](https://spec.openapis.org/oas/v3.0.3) contract for a multi-rescue dog adoption platform.

Rescue organisations can register themselves and add dogs available for adoption. Potential adopters can browse and filter dogs across all rescues and submit adoption requests.

## Domain Model

The platform is built around three core entities: **Rescue Organisations**, **Dogs**, and **Adoption Requests**.

```
Rescue  1 ──< Dogs  1 ──< Adoption Requests
```

A Rescue lists Dogs. Each Dog moves through a status lifecycle (`available → reserved → adopted`).
Adopters submit Adoption Requests against a Dog when its status is `available`.

See [`docs/domain-model.md`](./docs/domain-model.md) for the full entity descriptions, ER diagram, and Dog status state machine.

## Business Processes

Five end-to-end processes drive the platform:

1. **Rescue Registration** — a rescue signs up and receives a `rescueId`
2. **Dog Listing** — a rescue adds a dog, which becomes immediately discoverable
3. **Browse Dogs** — an adopter searches and filters dogs across all rescues
4. **Adoption Request** — an adopter submits a request; the dog moves to `reserved`
5. **Dog Update / Removal** — a rescue updates or removes a dog listing

Each process that writes data also publishes a domain event so downstream systems can react without polling the REST API.

See [`docs/business-processes.md`](./docs/business-processes.md) for sequence diagrams of each process.

## API Contract

The full API specification is defined in [`openapi.yaml`](./openapi.yaml).

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/rescues` | List all registered rescue organisations |
| `POST` | `/rescues` | Register a new rescue organisation |
| `GET` | `/rescues/{rescueId}` | Get details of a specific rescue |
| `GET` | `/rescues/{rescueId}/dogs` | List dogs available from a specific rescue |
| `POST` | `/rescues/{rescueId}/dogs` | Add a dog to a rescue |
| `GET` | `/dogs` | List all dogs across all rescues (supports filtering) |
| `GET` | `/dogs/{dogId}` | Get details of a specific dog |
| `PUT` | `/dogs/{dogId}` | Update a dog's details |
| `DELETE` | `/dogs/{dogId}` | Remove a dog from the platform |
| `POST` | `/dogs/{dogId}/adoptions` | Submit an adoption request for a dog |

### Filtering dogs

The `GET /dogs` and `GET /rescues/{rescueId}/dogs` endpoints support the following query parameters:

| Parameter | Type | Description |
|-----------|------|-------------|
| `breed` | string | Filter by breed (e.g. `Labrador`) |
| `maxAgeYears` | integer | Filter to dogs no older than this age |
| `status` | string | Filter by status: `available`, `reserved`, `adopted`, `unavailable` |
| `page` | integer | Page number (default: `1`) |
| `limit` | integer | Results per page (default: `20`, max: `100`) |

## Running Locally

You need [Docker](https://docs.docker.com/get-docker/) and [Docker Compose](https://docs.docker.com/compose/install/).

```bash
docker compose up
```

This starts two services:

| Service | URL | Description |
|---------|-----|-------------|
| **Docs website** | http://localhost:8080 | Landing page with features, quick start, and schemas |
| **API Reference** | http://localhost:8080/api.html | Full interactive Redoc API reference |
| **Prism mock server** | http://localhost:4010 | Mock server backed by the OpenAPI spec |

### Example mock requests

Once the mock server is running, you can try the API immediately without a real backend:

```bash
# List all rescues
curl http://localhost:4010/rescues

# List all dogs
curl http://localhost:4010/dogs

# List all dogs from a specific rescue
curl http://localhost:4010/rescues/a3bb189e-8bf9-3888-9912-ace4e6543002/dogs

# Get a specific dog
curl http://localhost:4010/dogs/f47ac10b-58cc-4372-a567-0e02b2c3d479

# Add a dog to a rescue (requires API key header)
curl -X POST http://localhost:4010/rescues/a3bb189e-8bf9-3888-9912-ace4e6543002/dogs \
  -H "Content-Type: application/json" \
  -H "X-API-Key: any-key" \
  -d '{
    "name": "Max",
    "breed": "Golden Retriever",
    "ageYears": 2,
    "sex": "male"
  }'

# Submit an adoption request
curl -X POST http://localhost:4010/dogs/f47ac10b-58cc-4372-a567-0e02b2c3d479/adoptions \
  -H "Content-Type: application/json" \
  -d '{
    "adopterName": "Jane Smith",
    "adopterEmail": "jane.smith@example.com",
    "adopterPhone": "+441987654321",
    "message": "I would love to give Buddy a forever home."
  }'
```

## Authentication

Write operations (`POST`, `PUT`, `DELETE`) require an `X-API-Key` header. The mock server accepts any non-empty value.

## Validate the spec

You can validate the OpenAPI specification using the [Spectral](https://stoplight.io/open-source/spectral) linter:

```bash
npx @stoplight/spectral-cli lint openapi.yaml
```

Or using the [Redocly CLI](https://redocly.com/docs/cli/):

```bash
npx @redocly/cli lint openapi.yaml
```