[![python](https://img.shields.io/badge/python-3.10+-success.svg)](https://devguide.python.org/versions)
[![license](https://img.shields.io/badge/license-MIT-success.svg)](https://opensource.org/licenses/MIT)

* [What is this project?](#what-is-this-project)
* [Quick demo](#quick-demo)
* [HOWTO](#howto)
  * [Using Docker](#using-docker) (for testing only)
  * [Using a venv](#using-a-venv)
* [Keyset pagination](#keyset-pagination)
* [API authentication](#api-authentication) (optional)

# What is this project?

A Python REST API on top of the FreeRADIUS database for automation purposes.

* It provides an [object-oriented view](https://github.com/angely-dev/pyfreeradius/blob/main/docs/conceptual-approach.md#class-diagram) of the database schema
* It implements [some logic](https://github.com/angely-dev/pyfreeradius/blob/main/docs/conceptual-approach.md#domain-logic) to ensure data consistency

It relies on [pyfreeradius](https://github.com/angely-dev/pyfreeradius) package which was originally embedded in this project.

> I recommend to read the docs of `pyfreeradius` since this project is now essentially a wrapper.

<img width="1292" height="1008" alt="476616451-ec229626-dff5-43ce-869b-0602b2454c56" src="https://github.com/user-attachments/assets/3ef759fa-551a-4572-ba5f-c10692ce791c" />

## What database support?

It works with MySQL, MariaDB, PostgreSQL and SQLite using DB-API 2.0 ([PEP 249](https://peps.python.org/pep-0249/)) compliant drivers such as:

```
mysql-connector-python
psycopg
psycopg2
pymysql
pysqlite3
sqlite3
```

It may work with other compliant drivers, yet not tested.

# Quick demo

## Get NASes, users and groups

Number of results is limited to `100` by default.

```sh
curl -X 'GET' http://localhost:8000/nas
#> 200 OK
[
    {"nasname": "3.3.3.3", "shortname": "my-super-nas", "secret": "my-super-secret"},
    {"nasname": "4.4.4.4", "shortname": "my-other-nas", "secret": "my-other-secret"},
    {"nasname": "4.4.4.5", "shortname": "another-nas", "secret": "another-secret"},
]
```
```sh
curl -X 'GET' http://localhost:8000/users
#> 200 OK
[
    {
        "username": "alice@adsl",
        "checks": [{"attribute": "Cleartext-Password", "op": ":=", "value": "alice-pass"}],
        "replies": [
            {"attribute": "Framed-IP-Address", "op": ":=", "value": "10.0.0.2"},
            {"attribute": "Framed-Route", "op": "+=", "value": "192.168.1.0/24"},
            {"attribute": "Framed-Route", "op": "+=", "value": "192.168.2.0/24"},
            {"attribute": "Huawei-Vpn-Instance", "op": ":=", "value": "alice-vrf"},
        ],
        "groups": [{"groupname": "100m", "priority": 1}],
    },
    {
        "username": "bob",
        "checks": [{"attribute": "Cleartext-Password", "op": ":=", "value": "bob-pass"}],
        "replies": [
            {"attribute": "Framed-IP-Address", "op": ":=", "value": "10.0.0.1"},
            {"attribute": "Framed-Route", "op": "+=", "value": "192.168.1.0/24"},
            {"attribute": "Framed-Route", "op": "+=", "value": "192.168.2.0/24"},
            {"attribute": "Huawei-Vpn-Instance", "op": ":=", "value": "bob-vrf"},
        ],
        "groups": [{"groupname": "100m", "priority": 1}],
    },
    # other users removed for brevity
    # …
]
```
```sh
curl -X 'GET' http://localhost:8000/groups
#> 200 OK
[
    {
        "groupname": "100m",
        "checks": [],
        "replies": [{"attribute": "Filter-Id", "op": ":=", "value": "100m"}],
        "users": [
            {"username": "bob", "priority": 1},
            {"username": "alice@adsl", "priority": 1},
            {"username": "eve", "priority": 1},
        ],
    },
    {
        "groupname": "200m",
        "checks": [],
        "replies": [{"attribute": "Filter-Id", "op": ":=", "value": "200m"}],
        "users": [{"username": "eve", "priority": 2}],
    },
    {
        "groupname": "250m",
        "checks": [],
        "replies": [{"attribute": "Filter-Id", "op": ":=", "value": "250m"}],
        "users": [],
    },
    # other groups removed for brevity
    # …
]
```

## Get a specific NAS, user or group

```sh
curl -X 'GET' http://localhost:8000/nas/3.3.3.3
#> 200 OK
{
    "nasname": "3.3.3.3",
    "shortname": "my-super-nas",
    "secret": "my-super-secret"
}
```
```sh
curl -X 'GET' http://localhost:8000/users/eve
#> 200 OK
{
    "username": "eve",
    "checks": [{"attribute": "Cleartext-Password", "op": ":=", "value": "eve-pass"}],
    "replies": [
        {"attribute": "Framed-IP-Address", "op": ":=", "value": "10.0.0.3"},
        {"attribute": "Framed-Route", "op": "+=", "value": "192.168.1.0/24"},
        {"attribute": "Framed-Route", "op": "+=", "value": "192.168.2.0/24"},
        {"attribute": "Huawei-Vpn-Instance", "op": ":=", "value": "eve-vrf"},
    ],
    "groups": [{"groupname": "100m", "priority": 1}, {"groupname": "200m", "priority": 2}],
}
```
```sh
#> 200 OK
curl -X 'GET' http://localhost:8000/groups/100m
{
    "groupname": "100m",
    "checks": [],
    "replies": [{"attribute": "Filter-Id", "op": ":=", "value": "100m"}],
    "users": [
        {"username": "bob", "priority": 1},
        {"username": "alice@adsl", "priority": 1},
        {"username": "eve", "priority": 1},
    ],
}
```

## Post a NAS, a user or a group

```sh
curl -X 'POST' \
  'http://localhost:8000/nas' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "nasname": "5.5.5.5",
  "shortname": "my-nas",
  "secret": "my-secret"
}'
#> 201 Created
```
```sh
curl -X 'POST' \
  'http://localhost:8000/users' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "username": "my-user@my-realm",
  "checks": [{"attribute": "Cleartext-Password", "op": ":=", "value": "my-pass"}],
  "replies": [
    {"attribute": "Framed-IP-Address", "op": ":=", "value": "192.168.0.1"},
    {"attribute": "Huawei-Vpn-Instance","op": ":=", "value": "my-vrf"}
  ]
}'
#> 201 Created
```
```sh
curl -X 'POST' \
  'http://localhost:8000/groups' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "groupname": "300m",
  "replies": [
    {"attribute": "Filter-Id", "op": ":=", "value": "300m"}
  ]
}'
#> 201 Created
```

## Patch a NAS, a user or a group

The update strategy follows [RFC 7396](https://datatracker.ietf.org/doc/html/rfc7396) (JSON Merge Patch) guidelines:

* omitted fields during the update are not modified
* `None` value means removal (i.e., resets a field to its default value)
* **a list field can only be overwritten (replaced)**

As a consequence of the last point, to add attributes to an existing user (or a group), you must fetch the existing attributes first, combine them with the new ones, and send the result as the update parameter.

```sh
curl -X 'PATCH' \
  'http://localhost:8000/nas/5.5.5.5' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "secret": "new-secret",
  "shortname": "new-shortname"
}'
#> 200 OK
```
```sh
curl -X 'PATCH' \
  'http://localhost:8000/users/my-user%40my-realm' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "checks": [{"attribute": "Cleartext-Password", "op": ":=", "value": "new-pass"}],
  "replies": [
    {"attribute": "Framed-IP-Address", "op": ":=", "value": "192.168.0.1"},
    {"attribute": "Framed-Route", "op": "+=", "value": "192.168.1.0/24"},
    {"attribute": "Huawei-Vpn-Instance","op": ":=", "value": "my-vrf"}
  ]
}'
#> 200 OK
```
```sh
curl -X 'PATCH' \
  'http://localhost:8000/groups/300m' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "replies": [
    {"attribute": "Filter-Id", "op": ":=", "value": "300m-premium"}
  ]
}'
#> 200 OK
```

## Delete a NAS, a user or a group

```sh
curl -X 'DELETE' http://localhost:8000/nas/5.5.5.5
#> 204 No Content
```
```sh
curl -X 'DELETE' http://localhost:8000/users/my-user@my-realm
#> 204 No Content
```
```sh
curl -X 'DELETE' http://localhost:8000/groups/300m
#> 204 No Content
```

Reminder:

* When a user is deleted, so are its attributes and its belonging to groups ([read more](https://github.com/angely-dev/pyfreeradius#delete-a-user))
* When a group is deleted, so are its attributes and its belonging to users ([read more](https://github.com/angely-dev/pyfreeradius#delete-a-group))

# HOWTO

**An instance of the FreeRADIUS server is NOT needed for testing.** The focus is on the FreeRADIUS database. As long as you have one, the API can run on a Python environment.

## Using Docker

I made a full Docker stack for testing purposes that should run "as is". It includes the API, a MariaDB database with the FreeRADIUS schema, and a FreeRADIUS server 😊

If you already have a FreeRADIUS database (either local or remote) or if you fear Docker, you can skip this section 🚀

```bash
wget https://github.com/angely-dev/freeradius-api/archive/refs/heads/master.zip
unzip master.zip
cd freeradius-api-master/docker
#
# For development environment (populates sample data):
docker compose up -d
#
# For production environment (no sample data):
# docker compose -f docker-compose.prod.yml up -d
```

Docker output should be like this:

```bash
Creating network "docker_default" with the default driver
Creating volume "docker_mariadb_data" with default driver
Pulling mariadb (mariadb:10.6)...
[…]
Pulling freeradius (freeradius/freeradius-server:3.2.3)...
[…]
Building radapi
[…]
Pulling myadmin (phpmyadmin:)...
[…]
Creating docker_mariadb_1 ... done
Creating docker_freeradius_1 ... done
Creating docker_myadmin_1 ... done
Creating docker_radapi_1  ... done
```

The services will be available at:
- API: http://localhost:8000
- phpMyAdmin: http://localhost:8000
- FreeRADIUS: UDP ports 1812 (authentication) and 1813 (accounting)

Then go to: http://localhost:8000/docs

## Environment-specific configurations

This project supports different environments to prevent development data from being populated in production:

1. **Development environment** (`docker-compose.yml`): Populates the database with sample data for testing
2. **Production environment** (`docker-compose.prod.yml`): Does not populate any sample data

To use the production environment, run:
```bash
docker compose -f docker-compose.prod.yml up -d
```

The environment is controlled by the `ENV` variable:
- `ENV=dev` (default): Populates development data
- `ENV=prod`: Does not populate development data

## Using a venv

## Using a venv

* Get the project and set the venv:

```bash
wget https://github.com/angely-dev/freeradius-api/archive/refs/heads/master.zip
unzip master.zip
cd freeradius-api-master
#
python3 -m venv venv
source venv/bin/activate
```

* Edit [`requirements.txt`](https://github.com/angely-dev/freeradius-api/blob/master/requirements.txt) to set the DB driver depending on your database system (MySQL, PostgreSQL, etc.):

```py
# Uncomment the appropriate line matching the DB-API 2.0 (PEP 249) compliant driver to use
mysql-connector-python
#psycopg
#psycopg2
#pymysql
#pysqlite3
#sqlite3
```

* Then install the requirements:

```bash
pip install -r requirements.txt
```

* Configure environment variables by copying and editing the `.env.example` file:

```bash
cp .env.example .env
# Edit .env with your database and API settings
```

Or set environment variables directly:

```bash
export DB_DRIVER="mysql.connector"
export DB_NAME="raddb"
export DB_USER="raduser"
export DB_PASS="radpass"
export DB_HOST="mydb"
export API_URL="http://localhost:8000"
export ITEMS_PER_PAGE="100"
```

Available environment variables:
- `DB_DRIVER`: Database driver (mysql.connector, psycopg2, sqlite3, etc.)
- `DB_NAME`: Database name
- `DB_USER`: Database username
- `DB_PASS`: Database password
- `DB_HOST`: Database host
- `DB_PORT`: Database port (optional)
- `API_URL`: Base URL for the API
- `API_PORT`: Port for the API server
- `ITEMS_PER_PAGE`: Items per page for pagination (default: 100)

* That's it! Now run the API and play with it live! All thanks to [FastAPI](https://github.com/tiangolo/fastapi) generating the OpenAPI specs which is rendered by [Swagger UI](https://github.com/swagger-api/swagger-ui) 😊

```bash
$ cd src
$ uvicorn api:app
INFO:     Started server process [12884]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
```

* Then go to: http://localhost:8000/docs

<img width="1292" height="1008" alt="476616451-ec229626-dff5-43ce-869b-0602b2454c56" src="https://github.com/user-attachments/assets/3ef759fa-551a-4572-ba5f-c10692ce791c" />

# Keyset pagination

As of [v1.3.0](https://github.com/angely-dev/freeradius-api/tree/v1.3.0), results are paginated (fetching all results at once is generally not needed nor recommended). There are two common options for pagination:

* Offset pagination (aka `LIMIT` + `OFFSET`) — not implemented
* **Keyset pagination (aka `WHERE` + `LIMIT`) — implemented**

In the era of infinite scroll, the latter is generally preferred over the former. Not only is it better at performance but also simpler to implement. [Read more about the implementation details.](https://github.com/angely-dev/pyfreeradius/blob/main/docs/keyset-pagination.md)

## Usage

Pagination is done through HTTP response headers (as per [RFC 8288](https://www.rfc-editor.org/rfc/rfc8288)) rather than JSON metadata. This is [debatable](https://news.ycombinator.com/item?id=12123511) but I prefer the returned JSON to contain only business data. Actually, this is what [GitHub API does](https://docs.github.com/en/rest/guides/using-pagination-in-the-rest-api#using-link-headers).

Here is an example for fetching usernames (the same applies for groupnames and nasnames):

```bash
$ curl -X 'GET' -i http://localhost:8000/users
HTTP/1.1 200 OK
date: Mon, 05 Jun 2023 20:07:09 GMT
server: uvicorn
content-length: 121
content-type: application/json
link: <http://localhost:8000/users?from_username=acb>; rel="next" # notice the Link header

["aaa","aab","aac","aad","aae","aaf","aag","aah","aai","aba","abb","abc","abd","abe","abf","abg","abh","abi","aca","acb"]
```

The API consumer (e.g., a frontend app) can then scroll to the next page `/users?from_username=acb`:

```bash
$ curl -X 'GET' -i http://localhost:8000/users?from_username=acb
HTTP/1.1 200 OK
date: Mon, 05 Jun 2023 20:07:43 GMT
server: uvicorn
content-length: 121
content-type: application/json
link: <http://localhost:8000/users?from_username=aed>; rel="next" # notice the Link header

["acc","acd","ace","acf","acg","ach","aci","ada","adb","adc","add","ade","adf","adg","adh","adi","aea","aeb","aec","aed"]
```

And so on until there are no more results to fetch:

```bash
$ curl -X 'GET' -i http://localhost:8000/users?from_username=zzz
HTTP/1.1 200 OK
date: Mon, 05 Jun 2023 20:08:06 GMT
server: uvicorn
content-length: 2
content-type: application/json
# no more Link header
```

> Only `rel="next"` is implemented since there wasn't a need yet for `rel="prev|last|first"`.

# API authentication

You may want to add authentication to the API.

## TL;DR

The API now supports API key authentication which can be enabled through environment variables.

To enable API key authentication:

1. Set `API_KEY_ENABLED=true` in your environment or `.env` file
2. Set `API_KEY=your-secret-key` in your environment or `.env` file
3. Optionally change the header name with `API_KEY_HEADER=X-API-Key` (default)

When enabled, all endpoints will require authentication via the specified header.

Example with default settings:
```sh
$ curl -X 'GET' -H 'X-API-Key: your-secret-key' -i http://localhost:8000/users
HTTP/1.1 200 OK

["bob","alice@adsl","eve","oscar@wil.de"]
```

> In the above code, we make use of both [global dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/global-dependencies/) and [security](https://fastapi.tiangolo.com/tutorial/security/first-steps/) features of FastAPI. API key is not properly documented yet but the issue https://github.com/tiangolo/fastapi/issues/142 provides some working snippets.

> The key would not normally go "in the clear" in the code like this. Depending on your setup, it would be passed using CI/CD variables for example.

## Demo

### Using `curl`

```sh
$ curl -X 'GET' -i http://localhost:8000/users
HTTP/1.1 403 Forbidden

{"detail":"Not authenticated"}
```
```sh
$ curl -X 'GET' -H 'X-API-Key: an-invalid-key' -i http://localhost:8000/users
HTTP/1.1 401 Unauthorized

{"detail":"Invalid key"}
```
```sh
$ curl -X 'GET' -H 'X-API-Key: my-valid-key' -i http://localhost:8000/users
HTTP/1.1 200 OK

["bob","alice@adsl","eve","oscar@wil.de"]
```

### Using the Web UI

Because the API key security scheme is part of the [OpenAPI specification](https://swagger.io/specification/#security-scheme-object), the need for authentication gets specified on the Web UI 😊

* You will now notice an `Authorize` button at the top-right corner as well as a little lock on each endpoint:

![swagger-auth-1](https://user-images.githubusercontent.com/4362224/214641998-2f323fb2-b90a-45a6-a09b-e4eadf1d1a29.png)

* To play with the API, you first need to provide the key by clicking that `Authorize` button:

![swagger-auth-2](https://user-images.githubusercontent.com/4362224/214642029-985ea676-bfe2-468f-bad8-c105b5cc5d69.png)

* Alternatively, on the Redoc Web UI:

![redoc-auth](https://user-images.githubusercontent.com/4362224/214642067-6c461a34-2679-43e2-88a7-b6316c459ae9.png)

## Explanation

FastAPI supports multiple security schemes, including OAuth2, API key and others. OAuth2 is a vast subject and will not be treated here. API key is a simple mechanism you probably already used in some projects. The [Swagger doc](https://swagger.io/docs/specification/authentication/api-keys/) explains it very well:

> **An API key is a token that a client provides when making API calls.** The key can be sent in the query string or as a request header or as a cookie. API keys are supposed to be a secret that only the client and server know. Like Basic authentication, **API key-based authentication is only considered secure if used together with other security mechanisms such as HTTPS/SSL.**

A request header is quite common. Some examples:

* `X-API-Key: <TOKEN>` (as per the Swagger doc and the one I used in the solution)
* `X-Auth-Token: <TOKEN>` (e.g., LibreNMS)
* `Authorization: Token <TOKEN>` (e.g., NetBox and [DjangoREST-based projects](https://github.com/encode/django-rest-framework/blob/3.14.0/rest_framework/authentication.py#L151-L161) more generally)
* `Authorization: Bearer <TOKEN>` (in accordance with [RFC 6750](https://www.rfc-editor.org/rfc/rfc6750#section-2.1) of the OAuth2 framework)

> The `X-` prefix denotes a non-standard HTTP header. You can set whatever name you prefer after that `X-`. The associated semantic is up to you. Some interesting background about the `X-` convention can be found in [RFC 6648](https://www.rfc-editor.org/rfc/rfc6648#appendix-A) which, by the way, officially deprecates it.

> Although `Authorization: Token` seems more standard as it follows [RFC 2617](https://www.rfc-editor.org/rfc/rfc2617) syntax, note the `Token` authentication scheme is not part of the [IANA HTTP Authentication Scheme Registry](https://www.iana.org/assignments/http-authschemes/http-authschemes.xhtml) so it is still custom in a sense. The only true standard header is `Authorization: Bearer` introduced by the OAuth2 framework.
