---
name: gen-http
description: >
  Generates .http files (Kulala / REST Client format) for Laravel debug controllers
  in app/Http/Controllers/Debug. Use this skill whenever the user asks to generate,
  create, or update an HTTP file for a debug controller, mentions "gen-http", wants
  to test a debug endpoint, or asks for a .http file for any controller under
  app/Http/Controllers/Debug. Also trigger when the user says things like "create
  the http file for X controller" or "generate requests for the debug endpoints".
---

# gen-http — HTTP File Generator for Debug Controllers

Generate `.http` files that follow the [Kulala HTTP file format](https://kulala.app/usage/http-file-format) for Laravel debug controllers under `app/Http/Controllers/Debug/`.

## What to do

Given a controller name, file path, or pattern, produce a ready-to-use `.http` file that covers every route registered for those controllers.

## Step-by-step process

### 1. Resolve the target controllers

The user may give you:
- A class name: `DebugCompanyListController`
- A file path: `app/Http/Controllers/Debug/DebugCompanyListController.php`
- A glob or keyword: `Debug/*`, `contacts`, `groove`
- Nothing explicit — if the current file open in the editor is a Debug controller, use it

Use `find` or `grep` on `app/Http/Controllers/Debug/` to locate the matching files.

### 2. Extract endpoints from routes/api.php

For each controller, grep `routes/api.php` for its class name to find every registered route. Capture:
- **HTTP method** (`GET`, `POST`, `PATCH`, `PUT`, `DELETE`)
- **URL path** (e.g., `/admin/debug/companies-list`)
- **Method name** (e.g., `__invoke`, `index`, `exportCsv`) — some controllers register multiple routes per method

Example grep:
```bash
grep -n "DebugCompanyListController" routes/api.php
```

### 3. Read each controller to extract parameters

Open the controller file and scan for:

**Query params (GET)** — look for:
- `$request->input('param_name', ...)` 
- `$request->filled('param_name')`
- `$request->query('param_name', ...)`
- `$request->validate([...])` keys when the method is GET

**Body fields (POST/PATCH)** — look for:
- `$request->validate(['field' => 'rule', ...])` → extract field names and rules
- `$request->input('field')` in non-GET contexts

**Route params** — note any `{id}`, `{cnpj}`, `{companyId}` etc. in the URL path.

Also look at docblocks (`/** @param ... */` and `Query params:` sections) for a concise param list.

### 4. Locate the output `http` folder

Before writing, find the folder that holds `.http` files. Look for a directory named `http` in this order:

1. `dev-docs/http/`
2. `docs/http/`
3. Any other `http/` directory at the project root or one level deep (e.g. use `find . -type d -name http -not -path '*/vendor/*' -not -path '*/node_modules/*'`).

Notes:
- This folder is often listed in `.gitignore`, so it may not show up in `git`-aware listings — check the filesystem directly with `find`/`ls`, not `git ls-files`.
- If exactly one `http/` folder is found, use it.
- If several are found, prefer `dev-docs/http/` then `docs/http/`, otherwise ask the user which to use.
- If none exists, create `dev-docs/http/` (falling back to `docs/http/` if a `docs/` directory already exists but `dev-docs/` does not), and mention where you placed the file.

### 5. Write the .http file

**Output file name:**
- Single controller → named after the controller in kebab-case, e.g. `debug-company-list.http`
- Multiple controllers → `debug.http` (or ask the user if unsure)
- Place the file inside the resolved `http/` folder from step 4.

**File structure:**

```http
@baseUrl = http://localhost:8000
@token = your-token-here

### <Endpoint title>
# <Short description of what it does>
# <If GET: list the filters/params it accepts>
<METHOD> {{baseUrl}}<path>[?param1=value&param2=value]
Authorization: Bearer {{token}}
[Content-Type: application/json]

[{
  "field1": value1,
  "field2": value2
}]

###
```

**Rules:**
- Start every file with the two variable declarations (`@baseUrl` and `@token`) followed by a blank line.
- Use `{{baseUrl}}` and `{{token}}` (double curly braces) throughout — never hardcode values.
- Separate each request block with `###` on its own line. A trailing `###` at the end is fine.
- For GET requests with query params: append them to the URL as `?param1=placeholder&param2=placeholder`. Use realistic placeholder values (e.g., `25` for `per_page`, `1` for `page`, `12345` for `id`).
- For POST/PATCH requests: add `Content-Type: application/json` header and a JSON body. Infer sensible placeholder values from field names and validation rules (e.g., an `integer` field gets `1`, a `string` field gets `"value"`).
- For routes with path params like `{id}`: use a `@<name> = 1` variable at the top and reference it as `{{<name>}}` in the URL.
- Add a `# Filters: param1, param2, ...` comment line for GET endpoints with multiple optional params.
- If a controller has multiple routes (e.g., `index` + `exportCsv`), include a block for each.
- Keep endpoint titles concise and human-readable (e.g., `List companies`, `Export CSV`, `Update plan subscription`).

## Example output

For `DebugCompanyListController` (GET `/admin/debug/companies-list`) and `DebugPlanSubscriptionUpdateController` (PATCH `/admin/debug/plan-subscription`):

```http
@baseUrl = http://localhost:8000
@token = your-token-here

### List companies
# Filters: id, cnpj, business_name, per_page, page
GET {{baseUrl}}/admin/debug/companies-list?per_page=25&page=1
Authorization: Bearer {{token}}

### Update plan subscription
PATCH {{baseUrl}}/admin/debug/plan-subscription
Authorization: Bearer {{token}}
Content-Type: application/json

{
  "plan_subscription_id": 1,
  "plan_id": 2
}
```

## Tips

- If a controller has no matching route in `routes/api.php`, note it with a comment: `# WARNING: no route registered for <ClassName>` and skip the block.
- If the same URL appears under both `index` and `__invoke`, deduplicate — only emit it once.
- For `exportCsv` routes: the response will be a file download, so add a comment `# Returns: CSV download`.
- When the user asks to update an existing `.http` file (e.g., adding a new controller), read the existing file first and append rather than overwrite, unless they ask for a full regeneration.
