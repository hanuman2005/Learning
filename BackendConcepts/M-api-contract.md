# Section M — API Contract & Docs

> Reference outline.

## M1 — REST design

**Diagram**
```
 GET    /orders        list      200
 POST   /orders        create    201
 GET    /orders/42     read      200 / 404
 PUT    /orders/42     replace   (idempotent)
 DELETE /orders/42     remove    (idempotent)
 nouns in URL, verbs in the HTTP method
```
**In plain words.** The URL names the thing. The method says what you do to it.

- Resources as nouns, correct HTTP verbs/status codes, consistent envelope (B7), idempotent PUT/DELETE.
- One line: nouns + correct verbs/status + consistent envelope.

## M2 — OpenAPI / schema

**Diagram**
```
 openapi.yaml (single source of truth)
   -> generated docs  -> generated client types  -> validation/tests
```
**In plain words.** Write the contract once and everything else is generated from it.

- Describe the API in **OpenAPI**; generate docs and client types; single source of truth for the
  contract.
- One line: define the contract in OpenAPI and generate docs/types from it.

## M3 — Versioning & compatibility

**Diagram**
```
 additive (new optional field)  = safe
 breaking (rename, remove, type change) = new version + deprecation
```
**In plain words.** Do not change what clients already rely on without warning.

- Additive changes stay compatible; breaking changes → new version + deprecation (J7/J8).
- One line: additive-by-default; version + deprecate breaking changes.

## M4 — Error contract

**Diagram**
```
 client sees : { success:false, error:{ code:"KYC_REJECTED", message:"..." } }
 server logs : full stack + SQL + req-id        (never sent to client)
```
**In plain words.** Tell the client what happened in a stable code. Keep the ugly details private.

- Stable machine-readable `error.code` (B7), documented error shapes, no stack traces/DB errors to
  clients (D3).
- One line: return stable documented error codes, never internal errors.

## M5 — Pagination / filtering contract

**Diagram**
```
 GET /orders?limit=20&cursor=abc&sort=created_at&status=PAID
 sort/filter from a whitelist, limit capped (max 100), documented
```
**In plain words.** Write down the rules for listing, so nobody guesses and nobody abuses it.

- Document pagination (offset vs cursor, B8), filter/sort params (whitelisted, B9), and limits.
- One line: document and whitelist pagination/filter/sort with caps.

## M6 — Docs & DX

**Diagram**
```
 code change -> regenerate docs -> examples -> published with the API version
```
**In plain words.** Docs that are out of date are worse than no docs.

- Keep docs generated + current; provide examples; version the docs with the API.
- One line: keep generated, example-rich docs in sync with the API.
