
# Database API

This is the API that talks to the database, instead of every service quering the database, the services use this API to make changes and read data off the database its the only service that has the database secrets and every component; it opens the sessions and closes them all talking to a postgreSQL database.

This file is a documentation of the different endpoints of the API

## Endpoints

### `POST /api/create-store`

Creates a new store.

**Request Body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | The name of the store |
| `address` | string | Yes | The address of the store |
| `manager` | string | Yes | The name of the manager |
| `store_id` | string | No | Specific ID for the store |
| `minimum_order_value` | number | No | Minimum order price (default: 10 GBP) |

**Responses**

`200 OK`
```json
{ "store_id": "100" }
```

`201 Conflict` — Store with the given ID already exists
```json
{ "error": "Store with the id {store_id} already exists" }
```

`400 Bad Request` — Missing required field(s)
```json
{ "error": "Missing a required fields: {fields}" }
```

### `POST /api/example`

Explain what endpoint does

**Request Body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `example` | type | Yes | Example description |
**Responses**

`200 OK`
```json
{ "example": "yes" }
```


`POST /api/<orderid>/mark-picked`

`POST /api/<storeid>/<routeid>/add-van {van: AA12 TST}`
