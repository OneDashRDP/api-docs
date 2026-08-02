# OneDash API

Base URL: `https://api.rdp-onedash.ru/api`

All identifiers, addresses, and keys used in the examples are fictitious.

## General rules

- Use HTTPS.
- Send POST request bodies as JSON with `Content-Type: application/json`.
- Every response is JSON.
- A successful response contains `"type": true`.
- An error contains `"type": false`, an `err` field, and usually a `message`.
- Check both the HTTP status code and the `type` field.
- Never write API keys or VPS passwords to logs.

## Authentication

Every `/api/*` method except `GET /api/ping` requires an API key:

```http
Authorization: Bearer YOUR_API_KEY
Accept: application/json
```

Example:

```bash
curl -sS \
  -H 'Accept: application/json' \
  -H 'Authorization: Bearer YOUR_API_KEY' \
  'https://api.rdp-onedash.ru/api/balance'
```

API keys are created and managed in the account dashboard. If an IP whitelist is configured for a key, requests are accepted only from allowed addresses.

A non-empty IP whitelist is mandatory for high-risk operations. This applies to VPS creation and cloning, credential retrieval, password changes, renewal, tariff changes, reinstallation, and VPS deletion.

## Response format

Successful response:

```json
{
  "type": true,
  "data": {}
}
```

Error response:

```json
{
  "type": false,
  "err": "wrong_parameters",
  "message": "Invalid request parameters"
}
```

Common HTTP status codes:

| Code | Meaning |
|---:|---|
| `200` | Request completed |
| `202` | Operation accepted and is being processed asynchronously |
| `401` | API key is missing, invalid, or disabled |
| `403` | Current IP is not allowed or an IP whitelist is required |
| `404` | Object or method not found |
| `409` | State conflict or another operation is already running |
| `422` | Invalid parameters or missing confirmation |
| `429` | Rate limit exceeded; honor the `Retry-After` header |
| `500` / `503` | Temporary service error |

Retry only safe requests. For POST operations, check the result of the previous request first to avoid duplicates.

## IP whitelist and operation confirmation

For sensitive operations:

1. Configure at least one allowed IP for the API key in the account dashboard.
2. Send the request from that IP.
3. When a method requires confirmation, include `"confirm": true`.

Missing confirmation returns the `confirmation_required` error.

## Service status

### GET /health

Public API gateway health check. Authentication is not required.

```bash
curl -sS 'https://api.rdp-onedash.ru/health'
```

```json
{
  "type": true,
  "service": "onedash-api-gateway",
  "status": "ready"
}
```

### GET /api/ping

Public API availability check.

```json
{
  "type": true,
  "service": "onedash-api",
  "version": 1,
  "time": 1893456000
}
```

## Authentication and reference data

### GET /api/auth/check

Validates the API key and access from the current IP.

```json
{
  "type": true,
  "data": {
    "authorized": true,
    "client_ip": "203.0.113.10"
  }
}
```

### GET /api/locations

Returns available locations and processor types.

```json
{
  "type": true,
  "data": {
    "locations": [
      {
        "location": "msk",
        "processors": ["intel", "amd"]
      }
    ],
    "synced_at": 1893456000
  }
}
```

Before creating a VPS, use only `location` and `processor` combinations present in this response.

### GET /api/systems

Returns available operating systems.

```json
{
  "type": true,
  "data": [
    {"code": "ubuntu_22", "name": "Ubuntu 22"},
    {"code": "windows_2022_en", "name": "Windows Server 2022 EN"}
  ]
}
```

Use the `code` value in creation and reinstallation requests.

### GET /api/tariffs

Returns tariffs for the selected location and processor.

Query parameters:

| Parameter | Required | Description |
|---|---:|---|
| `location` | no | Location code; defaults to `msk` |
| `processor` | no | Processor type; defaults to `intel` |

Example:

```http
GET /api/tariffs?location=msk&processor=intel
```

```json
{
  "type": true,
  "data": {
    "location": "msk",
    "processor": "intel",
    "currency": "RUB",
    "items": [
      {
        "id": 5,
        "name": "First",
        "config": {"cpu": 1, "ram": 1, "hard": 100},
        "prices": [
          {"period": 7, "price": 49, "discount": 0}
        ]
      }
    ]
  }
}
```

## Balance

### GET /api/balance

Returns the account balance.

```json
{
  "type": true,
  "data": {
    "balance": 100,
    "currency": "RUB",
    "locale": "ru"
  }
}
```

### POST /api/balance/top-up

Creates a link for manually adding funds to the balance.

Parameters:

| Parameter | Type | Description |
|---|---|---|
| `amount` | number | Amount from `1` to `1000000` |

```json
{
  "amount": 500
}
```

```json
{
  "type": true,
  "url": "https://pay.example.com/payment/EXAMPLE"
}
```

The balance changes only after the payment link has been paid successfully.

## Orders

### GET /api/orders

Returns orders belonging to the current account.

```json
{
  "type": true,
  "data": [
    {
      "order_id": 10001,
      "tariff": {"id": 5, "name": "First"},
      "location": "msk",
      "vps_list": [],
      "finish_time": {"epoch": 1893456000, "days_remaining": 30}
    }
  ]
}
```

### POST /api/orders/info

Returns information about one order.

```json
{
  "order_id": 10001
}
```

```json
{
  "type": true,
  "data": {
    "order_id": 10001,
    "tariff": {"id": 5, "name": "First"},
    "location": "msk",
    "vps_list": [],
    "finish_time": {"epoch": 1893456000, "days_remaining": 30}
  }
}
```

### POST /api/orders/renew

Renews an order and charges the account balance. An IP whitelist and `confirm=true` are required.

| Parameter | Type | Description |
|---|---|---|
| `order_id` | integer | Active order ID |
| `period` | integer | Renewal period in days |
| `confirm` | boolean | Must be `true` |

```json
{
  "order_id": 10001,
  "period": 30,
  "confirm": true
}
```

```json
{
  "type": true,
  "order_id": 10002
}
```

This method performs a one-time renewal. Automatic renewal is managed manually in the account dashboard and is not available through the API.

### POST /api/orders/{id}/tariff

Changes the tariff of an active order. An IP whitelist and `confirm=true` are required.

```json
{
  "tariff_id": 6,
  "confirm": true
}
```

A successful request returns HTTP `202`:

```json
{
  "type": true,
  "data": {
    "order_id": 10001,
    "old_tariff_id": 5,
    "new_tariff_id": 6,
    "new_tariff_name": "Second",
    "remaining_value_preserved": true,
    "lease_period_recalculated": true,
    "status": "changing"
  }
}
```

Request the order information again after completion.

## VPS

### GET /api/vps

Returns VPS instances belonging to the current account.

Query parameters:

| Parameter | Default | Limit |
|---|---:|---:|
| `page` | `1` | at least `1` |
| `per_page` | `50` | from `1` to `100` |

```json
{
  "type": true,
  "data": {
    "items": [
      {
        "id": 20001,
        "order_id": 10001,
        "name": "Production VPS",
        "active": true,
        "credentials_ready": true,
        "connection": {"host": "203.0.113.20", "port": 22},
        "os": "ubuntu_22",
        "location": "msk",
        "processor": "intel",
        "tariff": {"id": 5, "name": "First"},
        "expires_at": 1893456000,
        "options": {"backup": false, "static_ip": true, "nvme": false}
      }
    ],
    "pagination": {"page": 1, "per_page": 50, "total": 1, "pages": 1}
  }
}
```

### POST /api/vps/create

Creates VPS instances and charges the account balance. An IP whitelist and `confirm=true` are required.

| Parameter | Type | Description |
|---|---|---|
| `period` | integer | Rental period in days |
| `tariff_id` | integer | ID from `GET /api/tariffs` |
| `location` | string | Code from `GET /api/locations` |
| `processor` | string | Type from `GET /api/locations` |
| `system` | string | Code from `GET /api/systems` |
| `count` | integer | Number of VPS instances, from `1` to `10` |
| `additional_options.static_ip` | boolean | Dedicated IP |
| `additional_options.nvme` | boolean | NVMe storage |
| `additional_options.backup` | boolean | Backups |
| `confirm` | boolean | Must be `true` |

```json
{
  "period": 30,
  "tariff_id": 5,
  "location": "msk",
  "processor": "intel",
  "system": "ubuntu_22",
  "count": 1,
  "additional_options": {
    "static_ip": true,
    "nvme": false,
    "backup": true
  },
  "confirm": true
}
```

```json
{
  "type": true,
  "order_id": 10001
}
```

Creation is asynchronous. Wait for the VPS to appear as ready in `GET /api/vps` and check `credentials_ready`.

### POST /api/vps/clone

Clones a source VPS to 1–5 target VPS instances. An IP whitelist and `confirm=true` are required.

```json
{
  "source_vps_id": 20001,
  "target_vpses": [20002, 20003],
  "confirm": true
}
```

```json
{
  "type": true
}
```

Target VPS instances must belong to the same account and be eligible for the operation. Do not start other operations on them before cloning finishes.

### POST /api/vps/{id}/credentials

Returns VPS connection credentials. An IP whitelist is required. The request body may be an empty JSON object.

```json
{}
```

```json
{
  "type": true,
  "data": {
    "vps_id": 20001,
    "protocol": "ssh",
    "connection": {"host": "203.0.113.20", "port": 22},
    "login": "root",
    "password": "EXAMPLE_PASSWORD"
  }
}
```

This response contains secret data. Do not cache it or write the password to logs.

### POST /api/vps/{id}/rename

Changes the VPS display name. The maximum length is 40 characters. An empty name resets the custom name.

```json
{
  "name": "Production VPS"
}
```

```json
{
  "type": true,
  "data": {"vps_id": 20001, "name": "Production VPS"}
}
```

### GET /api/vps/{id}/power

Returns the power state and the last power command state.

```json
{
  "type": true,
  "data": {
    "vps_id": 20001,
    "power_state": "running",
    "power_action": "reboot",
    "power_action_status": "done",
    "busy": false,
    "stale": false
  }
}
```

### POST /api/vps/{id}/power

Submits a power command.

Allowed `action` values: `start`, `reboot`, `shutdown`.

```json
{
  "action": "reboot"
}
```

```json
{
  "type": true,
  "data": {
    "vps_id": 20001,
    "action": "reboot",
    "power_action_status": "queued"
  }
}
```

Poll `GET /api/vps/{id}/power` for completion.

### GET /api/vps/{id}/ptr

Returns the current PTR record and the state of the latest request.

```json
{
  "type": true,
  "data": {
    "vps_id": 20001,
    "ip": "203.0.113.20",
    "current_ptr": "host.example.com.",
    "last_request": null
  }
}
```

### POST /api/vps/{id}/ptr

Sets or removes a PTR record.

Set:

```json
{
  "ptr": "host.example.com"
}
```

Remove:

```json
{
  "delete": 1
}
```

A successful request returns HTTP `202`. Poll `GET /api/vps/{id}/ptr` for completion.

### POST /api/vps/{id}/password

Queues a password change. An IP whitelist is required.

Password requirements:

- 13 to 20 characters;
- at least one digit;
- at least one uppercase Latin letter;
- Latin letters, digits, and `!@#%^*_+=.,:?-` are allowed.

```json
{
  "password": "ExampleSecure9!"
}
```

A successful request returns HTTP `202`:

```json
{
  "type": true,
  "data": {
    "request_id": 30001,
    "vps_id": 20001,
    "status": "queued"
  }
}
```

### GET /api/vps/{id}/password/{request_id}

Returns the password change status. An IP whitelist is required.

Possible `status` values: `queued`, `processing`, `success`, `failed`.

```json
{
  "type": true,
  "data": {
    "request_id": 30001,
    "vps_id": 20001,
    "status": "success",
    "finished": true,
    "successful": true,
    "created_at": 1893456000
  }
}
```

### GET /api/vps/{id}/snapshots

Returns VPS snapshots and the state of related operations.

```json
{
  "type": true,
  "data": {
    "vps_id": 20001,
    "items": [],
    "count": 0,
    "limit": 3
  }
}
```

### GET /api/vps/{id}/backups

Returns backups. This method is available only when the backup option is enabled for the VPS.

```json
{
  "type": true,
  "data": {
    "vps_id": 20001,
    "enabled": true,
    "items": [],
    "count": 0
  }
}
```

### POST /api/vps/{id}/reinstall

Starts an OS reinstallation. All VPS data will be erased. An IP whitelist and `confirm=true` are required.

```json
{
  "system": "ubuntu_24",
  "confirm": true
}
```

A successful request returns HTTP `202`:

```json
{
  "type": true,
  "data": {
    "vps_id": 20001,
    "old_system": "ubuntu_22",
    "new_system": "ubuntu_24",
    "status": "reinstalling"
  }
}
```

After completion, refresh the VPS list and retrieve the credentials again. The VPS identifier in the list may change during reinstallation.

### GET /api/vps/{id}/delete-info

Calculates the refund before deletion. This method does not delete anything.

```json
{
  "type": true,
  "data": {
    "vps_id": 20001,
    "order_id": 10001,
    "refund_amount": 25,
    "currency": "ru",
    "days_left": 15,
    "early_delete": false,
    "early_fee": 0,
    "note": null
  }
}
```

Always show the user the current values from this response before deletion.

### POST /api/vps/{id}/delete

Permanently deletes the VPS and applies the applicable refund calculation. An IP whitelist and `confirm=true` are required.

```json
{
  "confirm": true
}
```

```json
{
  "type": true,
  "data": {
    "vps_id": 20001,
    "refund_amount": 25,
    "refund_amount_text": "25 RUB",
    "early_delete": false,
    "early_fee": 0,
    "status": "deleting"
  }
}
```

This operation is irreversible. Do not repeat the request after a successful response.

## Endpoint summary

| Method | Purpose |
|---|---|
| `GET /health` | Gateway health |
| `GET /api/ping` | API health |
| `GET /api/auth/check` | Authentication check |
| `GET /api/locations` | Locations and processors |
| `GET /api/systems` | Operating systems |
| `GET /api/tariffs` | Tariffs |
| `GET /api/balance` | Balance |
| `POST /api/balance/top-up` | Top-up link |
| `GET /api/orders` | Order list |
| `POST /api/orders/info` | Order information |
| `POST /api/orders/renew` | One-time renewal |
| `POST /api/orders/{id}/tariff` | Tariff change |
| `GET /api/vps` | VPS list |
| `POST /api/vps/create` | VPS creation |
| `POST /api/vps/clone` | VPS cloning |
| `POST /api/vps/{id}/credentials` | Connection credentials |
| `POST /api/vps/{id}/rename` | Rename VPS |
| `GET /api/vps/{id}/power` | Power state |
| `POST /api/vps/{id}/power` | Power command |
| `GET /api/vps/{id}/ptr` | PTR state |
| `POST /api/vps/{id}/ptr` | Change PTR |
| `POST /api/vps/{id}/password` | Change password |
| `GET /api/vps/{id}/password/{request_id}` | Password change status |
| `GET /api/vps/{id}/snapshots` | Snapshots |
| `GET /api/vps/{id}/backups` | Backups |
| `POST /api/vps/{id}/reinstall` | Reinstall OS |
| `GET /api/vps/{id}/delete-info` | Deletion calculation |
| `POST /api/vps/{id}/delete` | Delete VPS |

## Integration recommendations

- Store API keys in a secret manager or environment variable.
- Restrict keys with an IP whitelist and use a separate key for each integration.
- Set connection and read timeouts.
- Honor `Retry-After` when receiving `429`.
- Poll asynchronous operations at reasonable intervals.
- Show operation details to the user before charges or deletion.
- Refresh the VPS list after creation, cloning, and reinstallation.
- Never share API keys, passwords, or payment links with third parties.
