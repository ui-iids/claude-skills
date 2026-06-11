---
name: flask-apis
description: Design and build RESTful JSON APIs with Flask as the backend — resource modeling, correct HTTP verbs and status codes, request validation and response serialization (marshmallow / flask-smorest), a consistent JSON error contract, token/JWT auth, pagination, filtering, versioning, CORS, rate limiting, and auto-generated OpenAPI docs. Complements the `flask` skill (which covers server-rendered, template-based Flask); use this one when the Flask app returns JSON to clients rather than rendering HTML. Use when the user invokes `/flask-apis`, asks to "build a REST API in Flask", "add an API endpoint", "design a Flask backend", "serialize this model to JSON", "add JWT auth to my Flask API", "document my Flask API with OpenAPI/Swagger", or similar.
---

# Flask RESTful APIs

You are building a **JSON HTTP API** with Flask as the backend for clients
(SPAs, mobile apps, other services), not a server-rendered web app. The
output of every endpoint is data, not HTML. Optimize for a **predictable,
self-describing contract**: consistent shapes, correct status codes, machine-
readable errors, and validation at the boundary.

This skill is the API counterpart to the **`flask` skill**. Everything that
skill says about the **application-factory pattern, `extensions.py`, class-
based config, SQLAlchemy models, Flask-Migrate, and the test-client setup
still applies** — do not restate or relitigate it. Read it for those shared
foundations. This skill only covers what is *different* about an API: there
are no templates, no Jinja, no WTForms, no flash messages, and no session-
cookie login flow. Validation moves from WTForms to schemas; rendering moves
from templates to serialization; errors return JSON, not error pages.

## Use flask-smorest

**`flask-smorest` is the standard for this team — default to it for every
Flask API.** Built on marshmallow + apispec, it gives you request validation,
response serialization, pagination, a consistent error envelope, and
**automatic OpenAPI/Swagger docs** from one set of schemas. Build with it
unless the user explicitly asks for something else on a specific project.

Set the API up the flask-smorest way:

- Instantiate `Api` against the app in the factory (`api = Api(app)`),
  configuring `API_TITLE`, `API_VERSION`, and `OPENAPI_VERSION` in config.
- Use `flask_smorest.Blueprint` (not `flask.Blueprint`) so routes feed the
  OpenAPI spec, and `flask.views.MethodView` classes for resources.
- Declare I/O with `@blp.arguments(...)` / `@blp.response(...)` and paginate
  with `@blp.paginate()`. Let the extension own validation, serialization,
  and the error shape rather than hand-wiring them.

```python
# app/__init__.py  (inside create_app)
from flask_smorest import Api
app.config['API_TITLE'] = 'Items API'
app.config['API_VERSION'] = 'v1'
app.config['OPENAPI_VERSION'] = '3.0.3'
api = Api(app)
api.register_blueprint(items_blp)
```

**Plain Flask + raw `jsonify` is a fallback only** — reach for it solely when
the user wants minimal dependencies or it's a throwaway handful of endpoints.
If you go that route, you hand-wire validation, serialization, the error
contract, and OpenAPI yourself; the REST and contract rules below still apply.
Whichever you pick, **stay consistent** — never mix raw `jsonify` endpoints
with schema-driven ones in one codebase without a deliberate reason.

## REST design fundamentals (non-negotiable)

These hold regardless of stack.

- **Resources are nouns, plural, lowercase, hyphenated.** `/api/v1/blog-posts`,
  not `/api/v1/getBlogPosts` or `/api/v1/blogPost`. The HTTP verb carries the
  action; never put verbs in the path.
- **Map verbs to operations and let status codes carry the result:**

  | Verb | Path | Meaning | Success status |
  |------|------|---------|----------------|
  | `GET` | `/items` | list (paginated) | `200` |
  | `GET` | `/items/{id}` | fetch one | `200` |
  | `POST` | `/items` | create | `201` + `Location` header |
  | `PUT` | `/items/{id}` | full replace | `200` (or `204`) |
  | `PATCH` | `/items/{id}` | partial update | `200` |
  | `DELETE` | `/items/{id}` | remove | `204` (no body) |

- **Use status codes honestly.** `200/201/204` for success; `400` malformed,
  `401` unauthenticated, `403` authenticated-but-forbidden, `404` missing,
  `409` conflict (e.g. duplicate unique key), `422` semantically invalid
  payload (validation), `429` rate-limited; `500` only for *unexpected*
  server faults — never return `200` with an `{"error": ...}` body.
- **Nest only to express ownership, one level deep.** `/users/{id}/orders`
  is fine; `/users/{id}/orders/{oid}/items/{iid}/...` is a smell — prefer
  `/order-items/{iid}` with a filter. Deep nesting couples clients to your
  hierarchy.
- **`GET` and `DELETE` carry no request body. `GET` is safe and idempotent;
  `PUT`/`DELETE` are idempotent; `POST` is not.** Honor these — clients,
  proxies, and retries depend on them.
- **Version from day one.** Put `/api/v1` in the URL prefix. It is the
  cheapest insurance you will ever buy.

## Structure: API blueprints

Use the factory + blueprint structure from the `flask` skill, but blueprints
are organized **by resource** and registered under a versioned prefix. Keep
handlers thin — validate, call a service/model method, serialize, return.

```python
# app/api/v1/items/routes.py
items_bp = Blueprint('items', __name__)

@items_bp.route('', methods=['GET'])
def list_items():
    page = ItemQueryArgs().load(request.args)        # validate query params
    items = item_service.list(**page)                # business logic lives here
    return jsonify(ItemSchema(many=True).dump(items)), 200
```

```python
# app/__init__.py  (inside create_app)
from app.api.v1.items.routes import items_bp
app.register_blueprint(items_bp, url_prefix='/api/v1/items')
```

With **flask-smorest** the same idea uses `Blueprint` from `flask_smorest`
and `MethodView` classes so docs and validation come for free:

```python
from flask_smorest import Blueprint
from flask.views import MethodView

blp = Blueprint('items', __name__, url_prefix='/api/v1/items',
                description='Operations on items')

@blp.route('/')
class Items(MethodView):
    @blp.arguments(ItemQueryArgs, location='query')
    @blp.response(200, ItemSchema(many=True))
    @blp.paginate()
    def get(self, args, pagination_parameters):
        return item_service.list(**args)

    @blp.arguments(ItemCreateSchema)
    @blp.response(201, ItemSchema)
    def post(self, new_data):
        return item_service.create(new_data)
```

## Validation and serialization with schemas

**Validate every inbound payload at the boundary; never trust request data.**
Use marshmallow schemas. Keep **separate schemas for input and output** so you
can accept fields you never echo (passwords) and emit fields clients can't set
(`id`, timestamps).

```python
from marshmallow import Schema, fields, validate

class ItemSchema(Schema):                 # response / output
    id = fields.Int(dump_only=True)
    name = fields.Str(required=True)
    price = fields.Decimal(as_string=True, required=True)
    created_at = fields.DateTime(dump_only=True)

class ItemCreateSchema(Schema):           # request / input
    name = fields.Str(required=True, validate=validate.Length(min=1, max=120))
    price = fields.Decimal(required=True, validate=validate.Range(min=0))
```

- `dump_only` = server-controlled, output only. `load_only` = input only
  (passwords, write-only tokens). Use them deliberately — this is your
  primary defense against mass-assignment and accidental data leaks.
- Validate query params with their own schema too (pagination, filters,
  sort). Don't read `request.args` raw.
- For PATCH, load with `partial=True` so missing fields aren't errors.
- Serialize money as strings (`Decimal(as_string=True)`) and datetimes as
  ISO-8601 (marshmallow default). Don't ship floats for currency.

## The JSON error contract

Every error — validation, not-found, auth, server fault — must return the
**same JSON shape**. Pick one and enforce it app-wide via error handlers so
clients can parse failures uniformly.

```python
def error_response(status, message, *, errors=None):
    body = {'error': {'code': status, 'message': message}}
    if errors:
        body['error']['details'] = errors      # field-level validation errors
    return jsonify(body), status

@app.errorhandler(404)
def not_found(e):
    return error_response(404, 'Resource not found')

@app.errorhandler(422)            # marshmallow ValidationError, via webargs
def validation_error(e):
    return error_response(422, 'Validation failed', errors=e.data.get('messages'))

@app.errorhandler(Exception)      # last-resort 500
def internal_error(e):
    db.session.rollback()         # critical: keep the session usable
    current_app.logger.exception(e)
    return error_response(500, 'Internal server error')
```

- **Never leak stack traces or SQL** to clients. Log them server-side
  (`logger.exception`) and return a generic `500` message.
- **Always `db.session.rollback()` on the error path** — same rule as the
  `flask` skill, doubly important when there's no template render to mask a
  poisoned session.
- flask-smorest already produces a consistent error envelope and handles
  marshmallow errors; if you use it, register one custom handler for your own
  domain exceptions rather than reinventing the rest.
- Raise typed exceptions from services (`abort(404)` /
  `werkzeug.exceptions.Conflict`) and let handlers translate them — don't
  build `Response` objects deep in business logic.

## Authentication: tokens, not session cookies

APIs are typically stateless and cross-origin, so the template-skill's
Flask-Login cookie flow usually does **not** apply. Default to **bearer
tokens** (JWT via `flask-jwt-extended`, or opaque tokens you store and look
up).

```python
from flask_jwt_extended import jwt_required, get_jwt_identity, create_access_token

@auth_bp.route('/login', methods=['POST'])
def login():
    creds = LoginSchema().load(request.get_json())
    user = user_service.authenticate(creds['email'], creds['password'])
    if not user:
        return error_response(401, 'Invalid credentials')
    return jsonify(access_token=create_access_token(identity=str(user.id))), 200

@items_bp.route('/<int:item_id>', methods=['DELETE'])
@jwt_required()
def delete_item(item_id):
    item_service.delete(item_id, owner_id=get_jwt_identity())
    return '', 204
```

- **Hash passwords with `werkzeug.security`** (`generate_password_hash` /
  `check_password_hash`) — identical to the `flask` skill. Never store or log
  plaintext or raw tokens.
- Keep access tokens short-lived; use refresh tokens for longevity. Store the
  signing secret in an env var, never in code.
- Distinguish `401` (no/invalid credentials) from `403` (valid identity,
  insufficient permission). Enforce **object-level authorization** — check
  the caller actually owns the resource, not merely that they're logged in.
- If the client genuinely is a first-party browser SPA on the same site,
  cookie-based auth is acceptable — but then you **must** add CSRF protection
  and `SameSite` cookies. For third-party/mobile clients, use tokens.

## Pagination, filtering, sorting

**Never return an unbounded list.** Every collection endpoint paginates by
default with a capped page size.

- Offset/limit (`?page=2&per_page=50`) is simplest and fine for most CRUD.
  Cursor/keyset pagination scales better for large or fast-changing data —
  use it when offset pages get expensive or items shift between requests.
- Return pagination metadata alongside the data, e.g.
  `{"data": [...], "meta": {"page": 2, "per_page": 50, "total": 1234}}`.
  Keep this envelope consistent across all list endpoints. (flask-smorest's
  `@blp.paginate()` adds an `X-Pagination` header for you.)
- Expose filtering and sorting as query params validated by a schema
  (`?status=active&sort=-created_at`). Whitelist sortable/filterable fields —
  never interpolate client input into a query or `order_by` raw.
- Translate filters into SQLAlchemy query conditions; **use parameterized
  queries / the ORM**, never string-built SQL.

## Cross-cutting concerns

- **CORS:** if a browser on another origin calls the API, configure
  `flask-cors` explicitly — set allowed origins, methods, and headers to the
  minimum needed. Do **not** ship `CORS(app)` wide-open `*` to production,
  especially with credentials.
- **Rate limiting:** protect public and auth endpoints with `flask-limiter`.
  Return `429` with a `Retry-After` header when exceeded.
- **Content type:** require and return `application/json`. Use
  `request.get_json()` (or schema loading) and handle a missing/invalid body
  as `400`, not a `500`.
- **Idempotency & concurrency:** for `PUT`/`PATCH` on contended resources,
  consider `ETag`/`If-Match` optimistic concurrency to avoid lost updates.
- **Pagination/list caching:** `Cache-Control` and `ETag` headers let clients
  and proxies skip unchanged responses; add them for read-heavy endpoints.

## OpenAPI / Swagger documentation

A REST API should be self-documenting.

- With **flask-smorest**, docs are generated from your schemas and
  `MethodView` classes automatically. Configure `API_TITLE`, `API_VERSION`,
  and `OPENAPI_VERSION` in config and mount Swagger UI / ReDoc. Keep schema
  docstrings and field `metadata={'description': ...}` meaningful — they
  become the API docs.
- With plain Flask, wire up `apispec` + `apispec-webframeworks` (or
  `flasgger`) to emit an OpenAPI spec, or maintain it manually. Don't leave
  an API undocumented.

## Testing APIs

Reuse the test-client fixtures from the `flask` skill (`app`, `client` with
`config.Testing`, in-memory SQLite). For APIs, assert on **JSON bodies and
status codes**, not rendered HTML.

```python
def test_create_item(client, auth_header):
    resp = client.post('/api/v1/items',
                        json={'name': 'Widget', 'price': '9.99'},
                        headers=auth_header)
    assert resp.status_code == 201
    body = resp.get_json()
    assert body['name'] == 'Widget'
    assert 'id' in body

def test_create_item_validation_error(client, auth_header):
    resp = client.post('/api/v1/items', json={'price': '-1'}, headers=auth_header)
    assert resp.status_code == 422
    assert 'details' in resp.get_json()['error']
```

Cover, at minimum, per resource: happy-path CRUD, validation failure (`422`),
not-found (`404`), unauthenticated (`401`), forbidden (`403`), and that list
endpoints paginate and cap page size. Use `client.post(..., json=...)` so the
content-type is set correctly.

## Common pitfalls

1. **Verbs in URLs** (`/createItem`) — the verb is the HTTP method.
2. **Returning `200` for errors** with an error body — breaks clients that
   key off status codes.
3. **One schema for input and output** — leaks server fields and enables
   mass-assignment. Split them; use `dump_only`/`load_only`.
4. **Unbounded list endpoints** — always paginate with a max page size.
5. **Leaking exceptions** — register a catch-all `500` handler; log the
   detail, return a generic message.
6. **Forgetting `db.session.rollback()`** on the error path — poisons the
   session for the rest of the request lifecycle.
7. **Wide-open CORS in production** — scope origins explicitly.
8. **Building SQL or `order_by` from raw client input** — whitelist and
   parameterize; use the ORM.
9. **No versioning** — retrofitting `/v1` after clients exist is painful.
10. **Mixing JSON `jsonify` endpoints with template rendering** in one app
    without intent — keep the API surface pure JSON.
