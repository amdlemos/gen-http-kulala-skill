---
name: gen-http
description: >
  Generates .http files in the Kulala / REST Client format for Laravel API
  controllers. Use this skill whenever the user asks to generate, create, or
  update an .http file for a controller or endpoint, mentions "gen-http" or
  "kulala", wants to hand-test an API route, or says things like "create the
  http file for the UserController", "generate requests for these endpoints",
  or "make a .http file for this API". Works for any controller under
  app/Http/Controllers (including nested subfolders), not just one area of the app.
---

# gen-http — Kulala .http File Generator for Laravel APIs

Generate `.http` files that follow the [Kulala HTTP file format](https://kulala.app/usage/http-file-format) (also compatible with the VS Code REST Client) for controllers in a Laravel application.

## What to do

Given a controller name, file path, or keyword, produce a ready-to-use `.http` file that covers every route registered for those controllers, with the correct full URL, headers, query params, and request body.

## Step-by-step process

### 1. Resolve the target controller(s)

The user may give you:
- A class name: `UserController`, `InvoiceExportController`
- A file path: `app/Http/Controllers/Billing/InvoiceController.php`
- A glob or keyword: `Admin/*`, `invoice`, `report`
- Nothing explicit — if a controller file is currently open in the editor, use that

Search `app/Http/Controllers/` (including nested subfolders) to locate the matching files. Prefer the file/search tools; a quick fallback is:

```bash
find app/Http/Controllers -name '*Controller.php' | grep -i <keyword>
```

### 2. Find every route for each controller

Routes can live in several files. Search **all** files under `routes/` (e.g. `api.php`, `web.php`, and any custom files like `admin.php`), because a single project often splits routes across many files:

```bash
grep -rn "UserController" routes/
```

Capture, for each match:
- **HTTP method** — `GET`, `POST`, `PUT`, `PATCH`, `DELETE`
- **Route path** as written (e.g. `users/{id}`)
- **Controller method** — either the array form `[UserController::class, 'show']` or the invokable form `UserController::class` (which maps to `__invoke`)

A controller commonly registers several routes, so collect them all.

### 3. Reconstruct the full URL path

The path written on the `Route::` line is usually **not** the full URL. Two things can prepend segments in front of it:

1. **The route file's mount prefix.** In Laravel 10 and earlier this is set in `app/Providers/RouteServiceProvider.php`; in Laravel 11+ it's in `bootstrap/app.php` under `->withRouting(...)`. For example a file mounted with `->prefix('api')` puts every route in it under `/api/...`, and `->prefix('api/admin')` under `/api/admin/...`. Open that file and find the prefix that applies to the route file your controller's route lives in.
2. **Nested groups.** Walk upward from the matched line through any enclosing `Route::prefix('...')`, `Route::group(['prefix' => '...'], ...)`, or `Route::controller(...)->prefix(...)` blocks and prepend each prefix.

The full path is: `mount-prefix` + `group-prefixes` + `route-path`. Note any route parameters (`{id}`, `{slug}`, `{uuid}`) — you'll turn these into variables.

If you can't confidently determine a prefix, prefer running `php artisan route:list --path=<fragment>` to read the resolved URI directly, and fall back to a best guess with a `# TODO: verify path` comment rather than silently emitting a wrong URL.

### 4. Extract parameters from the controller

Open each controller method and scan for what the request needs:

**Query params (GET / DELETE)** — look for:
- `$request->input('name', $default)`, `$request->query('name')`, `$request->filled('name')`, `$request->boolean('name')`
- `$request->validate([...])` keys when the method reads them from the query string
- A `FormRequest` class type-hinted on the method — open it and read its `rules()`

**Body fields (POST / PUT / PATCH)** — look for:
- `$request->validate(['field' => 'rules', ...])` → field names and rules
- A type-hinted `FormRequest` → its `rules()` method
- `$request->input('field')` / `$request->only([...])`

**Route params** — the `{id}`-style segments found in step 3.

Docblocks (`@param`, "Query params:" notes) are a good shortcut when present.

### 5. Determine authentication

Check the middleware on the route or its enclosing group (e.g. `->middleware('auth:sanctum')`, `auth:api`, `->middleware(['auth', ...])`, or a group-level `middleware`). If the route is behind auth middleware, include an `Authorization` header. If it is clearly public (login, register, webhooks, health checks — typically no auth middleware), omit the auth header. When unsure, include it — it's the common case and easy for the user to delete.

### 6. Locate the output `http/` folder

Find the directory that holds `.http` files, in this order:

1. `dev-docs/http/`
2. `docs/http/`
3. Any other `http/` directory at the project root or one level deep:
   ```bash
   find . -type d -name http -not -path '*/vendor/*' -not -path '*/node_modules/*'
   ```

Notes:
- This folder is often git-ignored, so check the filesystem directly (`find`/`ls`), not `git ls-files`.
- If exactly one is found, use it. If several, prefer `dev-docs/http/` then `docs/http/`, otherwise ask which to use.
- If none exists, create `dev-docs/http/` (or `docs/http/` if a `docs/` dir already exists but `dev-docs/` doesn't) and mention where you placed the file.

### 7. Write the .http file

**Output file name:**
- Single controller → the controller name in kebab-case, e.g. `UserController` → `user.http`, `InvoiceExportController` → `invoice-export.http`
- Multiple controllers → a descriptive name for the group (e.g. `admin.http`), or ask if unsure
- Place it inside the resolved `http/` folder.

**File structure:**

```http
@baseUrl = http://localhost:8000
@token = your-token-here

### <Endpoint title>
# <Short description of what it does>
# <If GET: list the filters/params it accepts>
<METHOD> {{baseUrl}}<full-path>[?param1=value&param2=value]
Authorization: Bearer {{token}}
[Content-Type: application/json]

[{
  "field1": value1,
  "field2": value2
}]

###
```

**Rules:**
- Start the file with the `@baseUrl` and `@token` declarations, then a blank line.
- Use `{{baseUrl}}` and `{{token}}` (double curly braces) throughout — never hardcode the host or a real token.
- Separate each request block with `###` on its own line. A trailing `###` is fine.
- GET/DELETE query params: append as `?param1=placeholder&param2=placeholder` with realistic placeholders (`25` for `per_page`, `1` for `page`, `1` for an id).
- POST/PUT/PATCH: add `Content-Type: application/json` and a JSON body. Infer placeholder values from field names and validation rules (`integer`→`1`, `string`→`"value"`, `boolean`→`true`, `email`→`"user@example.com"`, `date`→`"2025-01-01"`, arrays→`[]`/`[1]`).
- Route params like `{id}`: declare a `@id = 1` variable near the top and reference it as `{{id}}` in the URL, so the user changes it in one place.
- For GET endpoints with several optional params, add a `# Filters: param1, param2, ...` comment.
- One block per registered route (e.g. `index`, `store`, `show`, `export`).
- Keep titles concise and human-readable (`List users`, `Create user`, `Export invoices CSV`).

## Example output

For a `UserController` with `GET /api/users` (paginated, filterable) and `POST /api/users` (create):

```http
@baseUrl = http://localhost:8000
@token = your-token-here

### List users
# Filters: search, role, per_page, page
GET {{baseUrl}}/api/users?per_page=25&page=1
Authorization: Bearer {{token}}

### Create user
POST {{baseUrl}}/api/users
Authorization: Bearer {{token}}
Content-Type: application/json

{
  "name": "value",
  "email": "user@example.com",
  "role_id": 1
}

###
```

## Tips

- If a controller has no matching route in `routes/`, note it with `# WARNING: no route registered for <ClassName>` and skip the block.
- If the same URL appears under both a named method and `__invoke`, deduplicate — emit it once.
- For file-download endpoints (CSV/PDF export), add `# Returns: file download` so the user knows the response isn't JSON.
- When asked to update an existing `.http` file (e.g. add a new controller), read it first and append rather than overwrite, unless the user asks for a full regeneration.
- Keep the file free of any real credentials or private data — placeholders only.
