# SPECS.md — org.barayin.wiki TiddlyWiki × atproto Hosting System

## Overview
This document specifies the architecture, schemas, protocols, and implementation details for hosting TiddlyWiki content on the atproto network under the namespace `org.barayin.wiki`.

---

## Namespace & Collections
- **Root namespace:** `org.barayin`
- **Wiki service namespace:** `org.barayin.wiki`
- **Collections:**
  - `org.barayin.wiki.wiki` — metadata record for a wiki
  - `org.barayin.wiki.tiddler` — per-tiddler record
  - `org.barayin.wiki.index` — title→rkey index per wiki
  - `org.barayin.wiki.tiddlerHistory` — revision log per tiddler

---

## Schema Definitions

### `org.barayin.wiki.wiki`
Metadata for a single wiki.
```json
{
  "slug": "string (record-key)",
  "name": "string",
  "about": "string?",
  "homeTitle": "string?",
  "theme": "string?",
  "visibility": "public|encrypted",
  "tiddlerCount": "int",
  "createdAt": "datetime",
  "updatedAt": "datetime"
}
````

### `org.barayin.wiki.tiddler`

A single tiddler record.

```json
{
  "wiki": "at-uri (org.barayin.wiki.wiki)",
  "title": "string",
  "type": "string (MIME)",
  "text": "string (≤128 KB) ?",
  "attachment": "blob?",
  "tags": "string[]",
  "fields": "array<kv>",
  "createdAt": "datetime",
  "modifiedAt": "datetime"
}
```

### `org.barayin.wiki.index`

Title→rkey index for a wiki.

```json
{
  "wiki": "at-uri",
  "updatedAt": "datetime",
  "version": "int",
  "items": [
    {
      "title": "string",
      "rkey": "string",
      "aliases": "string[]"
    }
  ]
}
```

### `org.barayin.wiki.tiddlerHistory`

Revision log for a tiddler.

```json
{
  "wiki": "at-uri",
  "tiddler": "at-uri (org.barayin.wiki.tiddler)",
  "truncated": "boolean",
  "items": [
    {
      "cid": "cid-link",
      "modifiedAt": "datetime",
      "note": "string?"
    }
  ]
}
```

### Shared Defs

* **kv:** `{ name: string, value: string }`
* **skinnyTiddler:** `{ title, type, tags[], modifiedAt }`

---

## AppView Endpoints

### `org.barayin.wiki.tiddler.list`

List skinny tiddlers for a wiki.

* Params: `{ wiki, limit?, cursor?, since?, tag?, titlePrefix? }`
* Returns: `{ tiddlers[], cursor? }`

### `org.barayin.wiki.tiddler.history`

Get revision history for a tiddler.

* Params: `{ wiki, rkey, limit?, cursor? }`
* Returns: `{ items[], cursor? }`

---

## Identifiers & Linking

* **Stable rkeys:** Derived from `wikiSlug:title-slug-HHHHHH` (title slug + 6-char base32 hash).
* **Linking:**

  * AT-URI without `cid` → always latest version.
  * AT-URI + `cid` → pinned version (strong reference).
* **Intra-wiki links:** Resolve `[[Title]]` via `index` table → rkey → fetch latest record.

---

## Revision Model

* Every edit replaces the record with CAS (`swapRecord`).
* Previous CIDs appended to `tiddlerHistory`.
* AppView provides history queries and diff support.

---

## Blob Handling

* Inline text ≤128 KB stays in `text`.
* Larger/binary content → `attachment` blob with `uploadBlob`.
* AppView generates thumbnails, strips metadata.

---

## Privacy Modes

* **public:** records visible to the network.
* **encrypted:** fields and blobs encrypted client-side with per-wiki symmetric key.

---

## SyncAdaptor (TiddlyWiki Plugin)

### Responsibilities

* `getSkinnyTiddlers` → call AppView `tiddler.list`.
* `loadTiddler` → fetch record via `getRecord`.
* `saveTiddler` → `putRecord` or `createRecord` with CAS.
* `deleteTiddler` → `deleteRecord`.
* Title→rkey cache built via `index` or scanning.

### Fields

* `revision`: last known CID (for CAS).
* `_attachmentBytes`, `_attachmentMime`: temporary fields for uploads.

---

## Server Components

### PDS

* Stores repo per user (DID).
* Validates records against lexicons.
* Exposes XRPC for `com.atproto.repo.*`.
* Emits firehose of commits.

### AppView

* Subscribes to firehose.
* Indexes tiddlers into Postgres/SQLite.
* Implements custom queries (`list`, `history`).
* Maintains backlinks, search, and title index.

---

## Federation

* Users control their own DID repos.
* AT-URIs resolve globally.
* AppViews federate content via firehose.
* Moderation handled at AppView layer.

---

## IPFS Integration (Optional)

### Commits

* Export repo deltas as CAR files.
* Import to IPFS (`ipfs dag import`).
* Pin commits via IPFS Cluster.

### Blobs

* On upload, push bytes to IPFS (`/api/v0/add`).
* Store returned CID in blob pointer.
* Serve blobs from PDS or IPFS gateway.

### Benefits

* Deduplication across repos.
* Long-term archival and pinning.
* Interop with IPFS tooling (CAR, DAG-CBOR).

---

## Deployment Model

### Minimal (dev)

* Single-host PDS + Postgres + Caddy TLS.
* SQLite AppView for `list` + `history`.
* IPFS node optional.

### Production

* Multi-tenant PDS cluster.
* Separate AppView cluster for indexing/search.
* IPFS pinning cluster for redundancy.
* Gateway (`wiki.barayin.org/@handle/<wiki>/<title>`).

---

## Backup & Restore

* Scheduled CAR exports of repos.
* Mirror blobs in object storage or IPFS.
* One-click restore into a fresh PDS.

---

## Developer Experience

* Lexicons published in `org.barayin.wiki`.
* TypeScript SDK generated via `lex-cli`.
* Helpers:

  * `rkey.ts`: slug/hash generator.
  * `cas.ts`: CAS wrappers.
  * `importer.ts`: resumable importer.
  * `indexer-client.ts`: AppView client.

---

## Moderation & Policy

* Per-wiki license string (default CC-BY-4.0).
* Abuse reporting and takedown workflow at AppView.
* Access controls via `visibility` and encrypted mode.

---

## Accessibility & Internationalization

* Alt text required for images.
* Unicode-safe slugs.
* Localizable UI strings.
* Bidirectional text support.
