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
| `415` | This method requires a JSON body |
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

## Repeated requests

Snapshot creation, rollback, and deletion, backup synchronization, restoration, and deletion, IP changes, location changes, processor changes, and additional option purchases require an IP whitelist, `"confirm": true`, and an `Idempotency-Key` header.

```http
Idempotency-Key: 71a6850d-5600-4b18-b184-c447eeac8f31
```

Use a separate key for each operation. Keys must contain 16 to 128 characters; Latin letters, digits, `.`, `_`, `:`, and `-` are allowed.

Repeating a request with the same key, method, ID, and parameters returns the stored response without executing the operation again. The `Idempotency-Replayed: true` header means that the response came from the journal. Keys apply across the account, including requests made with different API keys. They do not expire automatically.

Changing the parameters returns HTTP `409` and `idempotency_conflict`. If the previous request is still running, the response contains `operation_in_progress`. Wait for its result and repeat the request with the original key.

After `operation_outcome_unknown`, a connection failure, or a timeout, do not create a new key for the same operation. Check the VPS or order state and contact support if necessary. An operation with an unknown outcome is not retried automatically.

For these methods, send parameters only in the JSON body. The `confirm`, `include_ram`, `allow_data_loss`, and additional option flags must be JSON booleans, such as `true`, not the string `"true"` or the number `1`. Unknown parameters and `null` values are not accepted. Invalid parameters return HTTP `422`; other body formats return HTTP `415`.

These POST methods allow 20 requests per minute per API key and method, including retries. New read methods allow 60 requests per minute; purchase history allows 6. Exceeding the limit returns HTTP `429` with `Retry-After`.

Paid changes return `status=payment_required` and a `payment_url`. Complete the payment using this link. Creating a payment link does not mean the change has been completed and does not automatically charge the account balance. HTTP `202` with `status=accepted` means that the operation has been accepted; check the result through the VPS or order information.

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

### GET /api/balance/transactions

Returns the latest 50 balance transactions for the current account. No parameters are required.

```json
{
  "type": true,
  "data": {
    "items": [
      {
        "created_at": 1893456000,
        "direction": "plus",
        "amount": "100.00",
        "currency": "RUB",
        "info": "Balance top-up"
      }
    ],
    "limit": 50
  }
}
```

`created_at` is a Unix timestamp in seconds. `amount` is a string without a currency symbol. `direction` preserves the value from the account dashboard journal. This method returns balance transactions; request purchase history separately.

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

### GET /api/orders/history

Returns purchase history for the current account: orders, additional options, IP changes, location changes, and processor changes. The history may include unpaid purchases.

Query parameters:

| Parameter | Default | Limit |
|---|---|---|
| `from` | 29 days before `to` | Date in `YYYY-MM-DD` format |
| `to` | Current date | Date in `YYYY-MM-DD` format, no later than today |
| `page` | `1` | From `1` to `1000000` |
| `per_page` | `50` | From `1` to `100` |

Both dates are included in the range and use Moscow time. The maximum range is 31 calendar days. Send several requests for a longer period.

```json
{
  "type": true,
  "data": {
    "items": [
      {
        "id": "order_10001",
        "title": "VPS purchase",
        "status": "success",
        "status_text": "Successful",
        "amount": "300.00",
        "currency": "RUB",
        "created_at": 1893456000,
        "info": []
      }
    ],
    "pagination": {
      "total": 1,
      "per_page": 50,
      "current_page": 1,
      "last_page": 1,
      "from": 1893445200,
      "to": 1896037199
    }
  }
}
```

`pagination.from` and `pagination.to` are the requested period boundaries as Unix timestamps, not row numbers. If `page` exceeds the number of pages, the account dashboard returns the last page.

`status=success` means a successful purchase; `status=fail` means a purchase without confirmed payment. `info` contains additional details without HTML. Repeat purchase links from the account dashboard are not returned.

### POST /api/orders/{id}/location

Changes the location of an existing order. An IP whitelist, `confirm=true`, and `Idempotency-Key` are required.

```json
{
  "location": "ams",
  "confirm": true
}
```

Use a location code from `GET /api/locations`. Availability, order restrictions, and pricing follow the account dashboard rules.

If payment is required:

```json
{
  "type": true,
  "data": {
    "order_id": 10001,
    "status": "payment_required",
    "payment_url": "https://pay.example.test/payment/example"
  }
}
```

If no additional payment is required, the request returns HTTP `202` and `status=accepted`. After completion, refresh the order and VPS information.

### POST /api/orders/{id}/processor

Changes the processor type of an existing order. An IP whitelist, `confirm=true`, and `Idempotency-Key` are required.

```json
{
  "processor": "amd",
  "confirm": true
}
```

`processor` accepts `intel` or `amd`. Send the desired processor type. If it is already selected, the method returns HTTP `200`, `status=unchanged`, and the current `processor` value.

If payment is required, the response contains `status=payment_required` and `payment_url`. If no additional payment is required, the request returns HTTP `202` and `status=accepted`. Availability and restrictions follow the account dashboard rules.

### POST /api/orders/{id}/options

Adds a dedicated IP, NVMe storage, or backups to an existing order. An IP whitelist, `confirm=true`, and `Idempotency-Key` are required.

| Parameter | Type | Description |
|---|---|---|
| `static_ip` | boolean | Purchase a dedicated IP; defaults to `false` |
| `nvme` | boolean | Purchase NVMe storage; defaults to `false` |
| `backup` | boolean | Purchase backups; defaults to `false` |
| `confirm` | boolean | Must be `true` |

```json
{
  "static_ip": true,
  "nvme": false,
  "backup": true,
  "confirm": true
}
```

Set at least one option to `true`. This method adds options; `false` does not disable an existing service. The account dashboard calculates the price for the order and its remaining rental period. The response contains `status=payment_required` and `payment_url`.

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

### GET /api/vps/availability

Checks configuration availability before creating a VPS.

Query parameters:

| Parameter | Default | Description |
|---|---|---|
| `tariff_id` | — | Required tariff ID |
| `location` | `msk` | Location code from `GET /api/locations` |
| `processor` | `intel` | `intel` or `amd` |
| `system` | `ubuntu_24` | OS code from `GET /api/systems` |
| `count` | `1` | From `1` to `10` |
| `static_ip` | `0` | `0` or `1` |
| `nvme` | `0` | `0` or `1` |

```json
{
  "type": true,
  "data": {
    "available": 2,
    "max_available": 3,
    "requested": 2,
    "can_create": true,
    "reason": null
  }
}
```

This method uses the account dashboard availability check. Results may be cached for up to 30 seconds and do not reserve resources. Availability is checked again before purchase. An unknown or inactive location and processor combination returns `configuration_unavailable`.

### POST /api/vps/create

Creates VPS instances and charges the account balance. An IP whitelist and `confirm=true` are required.

| Parameter | Type | Description |
|---|---|---|
| `period` | integer | Rental period in days |
| `tariff_id` | integer | ID from `GET /api/tariffs` |
| `location` | string | Code from `GET /api/locations` |
| `processor` | string | Type from `GET /api/locations` |
| `system` | string | Code from `GET /api/systems` |
| `skip_desktop_link` | boolean | Do not create the OneDash “Services and Contacts” shortcut (`links.onedash.net`) on the Windows public desktop. Optional; defaults to `false` and has no effect on Linux |
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
  "system": "windows_10_ru",
  "skip_desktop_link": true,
  "count": 1,
  "additional_options": {
    "static_ip": true,
    "nvme": false,
    "backup": true
  },
  "confirm": true
}
```

In this example, the OneDash shortcut pointing to `links.onedash.net` is not created on the Windows desktop. Send `skip_desktop_link` as a JSON boolean (`true` or `false`), not as a string. If the parameter is omitted or set to `false`, the shortcut is created as usual.

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

### GET /api/vps/{id}/ips

Returns IP change options for a VPS belonging to the current account.

```json
{
  "type": true,
  "data": {
    "vps_id": 20001,
    "choices": [
      {"value": "203.0.113.21", "ip": "203.0.113.21"}
    ],
    "requires_recreate": false,
    "first_ip_change_free_available": false,
    "price": "100",
    "currency": "RUB"
  }
}
```

For a dedicated IP, `value` contains the address. For a shared IP, `value` contains the target node ID as a string, `ip` is the displayed address, and `requires_recreate=true` warns that the VPS will be recreated and its data lost. Send `value` in the IP change request.

The list and price are not reserved. Address availability and eligibility for a free change are checked again when the request is submitted.

### POST /api/vps/{id}/ip

Requests an IP change for an existing VPS. An IP whitelist, `confirm=true`, and `Idempotency-Key` are required.

```json
{
  "new_ip": "203.0.113.21",
  "confirm": true
}
```

`new_ip` is the string from the `value` field returned by `GET /api/vps/{id}/ips`. If `requires_recreate=true`, also send `"allow_data_loss": true`. Without this confirmation, the method returns `data_loss_confirmation_required`.

A paid change returns `status=payment_required` and `payment_url`. A free change may return `status=completed` or HTTP `202` with `status=accepted`. Refresh the VPS connection details after completion.

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

### POST /api/vps/{id}/snapshots

Creates a VPS snapshot. An IP whitelist, `confirm=true`, and `Idempotency-Key` are required.

| Parameter | Type | Description |
|---|---|---|
| `title` | string | Title up to 120 characters; defaults to an empty string |
| `description` | string | Description up to 500 characters; defaults to an empty string |
| `include_ram` | boolean | Save memory state; defaults to `false` |
| `confirm` | boolean | Must be `true` |

```json
{
  "title": "Before update",
  "description": "",
  "include_ram": false,
  "confirm": true
}
```

A successful request returns HTTP `202`:

```json
{
  "type": true,
  "data": {
    "vps_id": 20001,
    "snapshot_id": 30001,
    "status": "accepted"
  }
}
```

Check the snapshot state through `GET /api/vps/{id}/snapshots`. Creation limits and restrictions on concurrent operations are the same as in the account dashboard.

### POST /api/vps/{id}/snapshots/{snapshot_id}/rollback

Restores a VPS from its snapshot. An IP whitelist, `confirm=true`, and `Idempotency-Key` are required.

```json
{
  "confirm": true
}
```

Changes made after the snapshot was created will be lost. A successful request returns HTTP `202`, `vps_id`, `snapshot_id`, and `status=accepted`. Check the VPS and snapshot state before starting another operation.

### POST /api/vps/{id}/snapshots/{snapshot_id}/delete

Deletes a VPS snapshot. An IP whitelist, `confirm=true`, and `Idempotency-Key` are required.

```json
{
  "confirm": true
}
```

A successful request returns HTTP `202`, `vps_id`, `snapshot_id`, and `status=accepted`. Snapshot deletion is irreversible.

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

### POST /api/vps/{id}/backups/sync

Synchronizes the VPS backup list with the storage server. An IP whitelist, `confirm=true`, and `Idempotency-Key` are required.

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
    "status": "completed",
    "count": 2
  }
}
```

Synchronization updates the list of existing backups and does not create a new backup. Afterwards, request `GET /api/vps/{id}/backups`.

### POST /api/vps/{id}/backups/{backup_id}/restore

Restores a VPS from its backup. An IP whitelist, `confirm=true`, and `Idempotency-Key` are required.

```json
{
  "confirm": true
}
```

Current VPS data will be replaced with the backup data. A successful request returns HTTP `202`, `vps_id`, `backup_id`, and `status=accepted`. Check the VPS and backup state until the operation finishes.

### POST /api/vps/{id}/backups/{backup_id}/delete

Deletes a VPS backup. An IP whitelist, `confirm=true`, and `Idempotency-Key` are required.

```json
{
  "confirm": true
}
```

A successful request returns HTTP `202`, `vps_id`, `backup_id`, and `status=accepted`. Backup deletion is irreversible.

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
| `POST /api/vps/{id}/snapshots` | Create snapshot |
| `POST /api/vps/{id}/snapshots/{snapshot_id}/rollback` | Roll back to snapshot |
| `POST /api/vps/{id}/snapshots/{snapshot_id}/delete` | Delete snapshot |
| `POST /api/vps/{id}/backups/sync` | Synchronize backups |
| `POST /api/vps/{id}/backups/{backup_id}/restore` | Restore backup |
| `POST /api/vps/{id}/backups/{backup_id}/delete` | Delete backup |
| `GET /api/vps/{id}/ips` | IP change options |
| `POST /api/vps/{id}/ip` | Change IP |
| `POST /api/orders/{id}/location` | Change location |
| `POST /api/orders/{id}/processor` | Change processor |
| `POST /api/orders/{id}/options` | Additional options |
| `GET /api/vps/availability` | Configuration availability |
| `GET /api/balance/transactions` | Balance transactions |
| `GET /api/orders/history` | Purchase history |

## Integration recommendations

- Store API keys in a secret manager or environment variable.
- Restrict keys with an IP whitelist and use a separate key for each integration.
- Set connection and read timeouts.
- Honor `Retry-After` when receiving `429`.
- Poll asynchronous operations at reasonable intervals.
- Show operation details to the user before charges or deletion.
- Refresh the VPS list after creation, cloning, and reinstallation.
- Never share API keys, passwords, or payment links with third parties.
