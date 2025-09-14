# SPECS.md — org.barayin.wiki TiddlyWiki × atproto Hosting System

## Overview
This specification defines the complete architecture, schemas, editorial authority, moderation, federation, and IPFS integration for hosting TiddlyWiki content on the atproto network under the namespace `org.barayin.wiki`.

---

## Namespace & Collections
- **Root namespace:** `org.barayin`
- **Wiki service namespace:** `org.barayin.wiki`

### Collections
- `org.barayin.wiki.wiki` — wiki metadata
- `org.barayin.wiki.tiddler` — per-tiddler content
- `org.barayin.wiki.index` — title→rkey index per wiki
- `org.barayin.wiki.tiddlerHistory` — revision log per tiddler
- `org.barayin.wiki.roles` — per-wiki role bindings
- `org.barayin.wiki.editorialPolicy` — default editorial authority for a wiki
- `org.barayin.wiki.publishTip` — authority-selected published revision for a tiddler
- `org.barayin.wiki.moderation.report` — abuse reports
- `org.barayin.wiki.moderation.action` — authority actions
- `org.barayin.wiki.changeProposal` — collaborative change proposals

### AppView Queries
- `org.barayin.wiki.tiddler.list`
- `org.barayin.wiki.tiddler.history`
- `org.barayin.wiki.moderation.state`
- `org.barayin.wiki.search`
- `org.barayin.wiki.backlinks`

---

## Core Data Schemas

### `org.barayin.wiki.wiki`
```json
{
  "slug": "string",
  "name": "string",
  "about": "string?",
  "homeTitle": "string?",
  "theme": "string?",
  "visibility": "public|encrypted",
  "tiddlerCount": "int",
  "editorialPolicy": "at-uri?",
  "createdAt": "datetime",
  "updatedAt": "datetime"
}
````

### `org.barayin.wiki.tiddler`

```json
{
  "wiki": "at-uri",
  "title": "string",
  "type": "string",
  "text": "string?",
  "attachment": "blob?",
  "tags": "string[]",
  "fields": "array<kv>",
  "createdAt": "datetime",
  "modifiedAt": "datetime"
}
```

### `org.barayin.wiki.index`

```json
{
  "wiki": "at-uri",
  "updatedAt": "datetime",
  "version": "int",
  "items": [
    { "title": "string", "rkey": "string", "aliases": "string[]" }
  ]
}
```

### `org.barayin.wiki.tiddlerHistory`

```json
{
  "wiki": "at-uri",
  "tiddler": "at-uri",
  "truncated": "boolean?",
  "items": [
    { "cid": "cid-link", "modifiedAt": "datetime", "note": "string?" }
  ]
}
```

### `org.barayin.wiki.roles`

```json
{
  "wiki": "at-uri",
  "updatedAt": "datetime",
  "roles": [
    { "role": "owner|editor|moderator|viewer", "did": "string" }
  ]
}
```

### `org.barayin.wiki.editorialPolicy`

```json
{
  "wiki": "at-uri",
  "authority": {
    "mode": "self|delegated|community",
    "authorityDid": "string?",
    "labelers": ["string"]?,
    "notes": "string?"
  },
  "defaults": {
    "autoPublishLatest": "boolean",
    "allowUserAuthorityOverride": "boolean"
  },
  "createdAt": "datetime",
  "updatedAt": "datetime"
}
```

### `org.barayin.wiki.publishTip`

```json
{
  "wiki": "at-uri",
  "tiddler": "at-uri",
  "target": { "uri": "at-uri", "cid": "cid-link" },
  "reason": "string?",
  "expiresAt": "datetime?",
  "actor": "did",
  "createdAt": "datetime",
  "updatedAt": "datetime"
}
```

### `org.barayin.wiki.moderation.report`

```json
{
  "target": "at-uri",
  "reason": "string",
  "category": "spam|abuse|copyright|malware|nsfw|other",
  "reporter": "did",
  "createdAt": "datetime",
  "context": "string?"
}
```

### `org.barayin.wiki.moderation.action`

```json
{
  "target": "at-uri|did|wiki-at-uri",
  "scope": "record|wiki|actor",
  "action": "hide|shadowHide|label|freeze|publishTip|unpublish|ban|unban",
  "labels": ["string"]?,
  "publishTip": { "uri": "at-uri", "cid": "cid-link" }?,
  "duration": "string?",
  "reason": "string?",
  "actor": "did",
  "createdAt": "datetime"
}
```

### `org.barayin.wiki.changeProposal`

```json
{
  "wiki": "at-uri",
  "title": "string",
  "proposer": "did",
  "changes": [
    {
      "type": "edit|create|delete|rename",
      "tiddler": "at-uri?",
      "rkey": "string?",
      "from": { "uri": "at-uri", "cid": "cid-link" }?,
      "to": { "title": "string?", "type": "string?", "text": "string?", "attachment": "blob?" }?
    }
  ],
  "note": "string?",
  "createdAt": "datetime",
  "status": "open|accepted|rejected|merged",
  "decider": "did?"
}
```

---

## Identifiers & Linking

* **rkeys:** `<wikiSlug>:<slug(title)>-<BASE32_6HASH>` ≤200 chars.
* **Latest:** `at://<did>/org.barayin.wiki.tiddler/<rkey>`
* **Pinned:** `{ uri, cid }`
* **Published view:** resolve via `publishTip` per authority.
* **Intra-wiki links:** use `index` to resolve title → rkey → latest/published.

---

## Editorial Authority

### Models

* **Self:** wiki owner is authority.
* **Delegated:** explicit DID as authority.
* **Community:** labeler sets or multisig.

### User Selection

* Clients may override authority if allowed.
* Authority discovery via DID doc service entry `atproto:appview`.

### Publish Control

* **Soft revert:** set `publishTip` to older revision.
* **Hard revert:** create new record with old content.
* **Freeze:** authority prevents new published tips.

---

## Moderation

### Reports

Anyone may create `moderation.report`.

### Actions

Authorities create `moderation.action`.
Precedence: `ban > hide/shadowHide > freeze > publishTip > autoPublish`.

### Effective State

Returned by `moderation.state` query:

```json
{
  "published": { "uri": "at-uri", "cid": "cid-link" }?,
  "visibility": "visible|hidden|shadowHidden",
  "labels": ["string"],
  "frozen": "boolean"
}
```

### Appeals

Represented as `report` with category `"appeal"`.

---

## Change Proposals

* Contributors propose edits via `changeProposal`.
* Owners/editors merge, creating standard tiddler records.
* Status transitions: `open → accepted|rejected|merged`.

---

## AppView Implementation

### Tables

* `tiddlers`, `tiddler_history`, `publish_tips`, `mod_actions`, `reports`, `title_index`.

### Queries

* **list:** skinny tiddlers, sorted `modifiedAt DESC`.
* **history:** full revision chain.
* **moderation.state:** effective publish + visibility per authority.
* **search:** tokenized full-text over title, text, tags.
* **backlinks:** link/transclusion graph from parsing wikitext.

### Pagination

* Cursor-based, opaque string.
* Max 100 per page.

---

## Encryption (encrypted visibility)

* **Algo:** AES-256-GCM, random 96-bit IV.
* **Key derivation:** Argon2id from passphrase; key-wrap for collaborators.
* **Fields:** store `{ alg, iv, salt, tag }` in record.
* **Blobs:** ciphertext stored, decrypted client-side.

---

## Blobs

* Inline text ≤128 KB; else `attachment`.
* Allowed MIME: text/*, image/*, video/\*, application/pdf, application/json.
* Thumbnails: generated at AppView for image/video.
* Size limit: 50 MB per blob.
* Deref failure: retry/backoff, mark with label.

---

## IPFS Integration

### Commits

* Mirror repo commits as CAR files.
* Import into IPFS with `ipfs dag import`.
* Pin via IPFS Cluster (replication factor configurable).

### Blobs

* Upload to IPFS via Kubo RPC (`/api/v0/add`).
* Store returned CID in PDS blob pointer.
* Serve via PDS or IPFS gateway.

---

## Deployment

### Minimal

* Single-host PDS, Postgres/SQLite AppView, optional IPFS node.

### Production

* Multi-tenant PDS cluster.
* Dedicated AppView cluster.
* IPFS pinning cluster.
* Gateway routes:

  * `/@handle/<wiki>/<title>`
  * `?cid=` for pinned version
  * `?authority=<did>` for authority view

---

## Backup & Restore

* Nightly CAR exports.
* Blob mirrors in IPFS or object store.
* One-click restore into PDS.

---

## Developer Experience

* Lexicons published under `org.barayin.wiki`.
* TypeScript SDK generated via `lex-cli`.
* Helpers: `rkey.ts`, `cas.ts`, `importer.ts`, `indexer-client.ts`.

---

## Observability & Quotas

* Metrics: repo ops, firehose lag, query latency.
* Rate limits: per-DID, with exponential backoff.
* Quotas: max tiddler count per wiki, max blob size.

---

## Security, Safety, A11y, i18n

* HTML sanitization, CSP on gateway.
* Alt-text required for images.
* Unicode-safe slugs, bidi-safe rendering.
* Localized UI strings.
* Strong CORS isolation on blob endpoints.

---

## Non-Goals

* CRDTs, real-time collaborative editing.
* Server-side plugin execution.
* Binary diff storage.
