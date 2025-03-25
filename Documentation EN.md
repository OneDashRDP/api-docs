# OneDash.RDP API

To work with the API, GET/POST requests are used:

    https://rdp-onedash.com/web-api/{method}

All parameters are passed in JSON format (Content-Type: application/json).

The response to any request will be a JSON string containing the "type" parameter.

An example of a successful request:

    {"type": true}
    
Example of a failed request:

    {"type": false}

# Authorization

In the body of each request, the client's API key must be present in the "Api-Key" header:

    Api-Key: 438b8af15a130133803f0ed0a2ae7d97d58b4605


# Test request
	 
**GET**

**https://rdp-onedash.com/web-api/test-request**

Example of a successful response:

    {"type": true}

# List of available OS
	 
**GET**

**https://rdp-onedash.com/web-api/systems-list**

Example of a successful response:

    {"type": true, "data": ["windows_10_ru", "windows_10_en", "windows_11_ru", "windows_11_en"]}
 
# Account balance
	 
**GET**

**https://rdp-onedash.com/web-api/balance**

Example of a successful response:

    {"type": true, "data": {"balance": 100, "currency": "USD"}}

# List of tariffs
	 
**GET**

**https://rdp-onedash.com/web-api/tariffs**

Example of a successful response:

    {"type":true,"data":[{"id":5,"name":"First","msk_prices":[{"price":49,"discount":0,"period":7},{"price":59,"discount":15,"period":10}],"ams_prices":[{"price":129,"discount":0,"period":7},{"price":159,"discount":15,"period":10}],"currency":"USD","config_info":{"cpu":1,"ram":1,"hard":100}}]}

`"msk_prices"` - prices in Moscow location

`"ams_prices"` - prices in Amsterdam location

Prices are indicated for each rental period separately.

# List of all orders
	 
**GET**

**https://rdp-onedash.com/web-api/all-orders**

Example of a successful response:

    {"type":true,"data":[{"order_id":19769,"tariff":{"id":5,"name":"First"},"location":"msk","vps_list":[{"id":12970,"os":"ubuntu_20","vps_ip":"123.123.123.13","vps_status":"runned"}],"finish_time":{"epoch":1886424023,"days_remaining":1665,"date":"11.10.2029 17:40"}}]}

The rental finish date is indicated in Moscow time zone.

Possible values ​​for the `"vps_status"` parameter:

`"runned"` - the server is up and running

`"not_runned"` - server in the process of starting

`"cloning"` - server in the process of cloning

# Create a vps
	 
The method creates a VPS by debiting the required amount from the account balance.

**POST**

**https://rdp-onedash.com/web-api/create-vps**

Possible parameters:

| Parameter | Description |
|-------------|-------------|
| `period`    | rental period: from 7 to 360 days    |
| `tariff_id`    | tariff id: [list of tariffs](#list-of-tariffs)    |
| `location`    | server location: `"msk"` `"ams"`   |
| `system`    | operation system: [list of OS](#List-of-available-OS)    |
| `count`    | Number of VPS: from 1 to 10 pcs.    |
| `additional_options["static_ip"]`    | additional option Static IP: true/false (default false)    |
| `additional_options["nvme"]`    | additional option NVME disk: true/false (default false)    |
| `additional_options["backup"]`    | additional option backup: true/false (default false)   |


Possible errors:

| Error | Description |
|-------------|-------------|
| `wrong_auth`    | authorization failed    |
| `wrong_parameters`    | one of the parameters is specified incorrectly   |
| `no_resources`    | not enough resources for the tariff and additional options   |
| `no_balance`    | not enough balance|

Example of a successful response:

    {"type":true,"order_id": 10000}

# Order information
	 
**POST**

**https://rdp-onedash.com/web-api/order-info**

Possible parameters:

| Parameter | Description |
|-------------|-------------|
| `order_id`    | order id    |

Example of a successful response:

    {"type":true,"data":{"order_id":10000,"tariff":{"id":5,"name":"First"},"location":"msk","vps_list":[{"id":10000,"os":"ubuntu_20","vps_ip":"123.123.123.13","vps_status":"runned"}],"finish_time":{"epoch":1886424023,"days_remaining":1662,"date":"11.10.2029 17:40"}}}

The rental finish date is indicated in Moscow time zone.

Possible values ​​for the `"vps_status"` parameter:

`"runned"` - the server is up and running

`"not_runned"` - server in the process of starting

`"cloning"` - server in the process of cloning

# Change of tariff
	 
**POST**

**https://rdp-onedash.com/web-api/change-tariff**

Possible parameters:

| Parameter | Description |
|-------------|-------------|
| `order_id`    | order id    |
| `new_tariff_id`    | new tariff id: [list of tariffs](#List-of-tariffs)     |

Example of a successful response:

    {"type":true}

# Reinstalling the OS
	 
**POST**

**https://rdp-onedash.com/web-api/reinstall-system**

Possible parameters:

| Parameter | Description |
|-------------|-------------|
| `vps_id`    | id vps: [list of all orders](#list-of-all-orders)    |
| `system`    | new OS: [list of available OS](#List-of-available-OS)     |

Example of a successful response:

    {"type":true}

# Reboot VPS
	 
**POST**

**https://rdp-onedash.com/web-api/restart-vps**

Possible parameters:

| Parameter | Description |
|-------------|-------------|
| `vps_id`    | id vps: [list of all orders](#list-of-all-orders)    |

Example of a successful response:

    {"type":true}

# Cloning of VPS

**POST**

**https://rdp-onedash.com/web-api/cloning-vps**

Possible parameters:

| Parameter | Description |
|-------------|-------------|
| `source_vps_id`    | vps source id: [list of all orders](#list-of-all-orders)    |
| `target_vpses`    | array of target servers (their IDs, up to 5): [list of all orders](#list-of-all-orders)    |

Example of a successful response:

    {"type":true}

# Order extension

The method extends the order by debiting the required amount from the account balance.
	 
**POST**

**https://rdp-onedash.com/web-api/renew-order**

Possible parameters:

| Parameter | Description |
|-------------|-------------|
| `order_id`    | order id: [list of all orders](#list-of-all-orders)    |
| `period`    | extension period (from 7 to 360 days)    |


Example of a successful response:

    {"type":true}

# Balance replenishment

The method generates a payment link to top up the account balance.
	 
**POST**

**https://rdp-onedash.com/web-api/top-up-balance**

Possible parameters:

| Parameter | Description |
|-------------|-------------|
| `amount`    | replenishment amount   |


Example of a successful response:

    {"type":true, "url": "https://pay.rdp-onedash.com/pay/1234567890"}

