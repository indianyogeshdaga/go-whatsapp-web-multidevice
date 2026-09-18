# Data and Media Management Reference

This document describes the data and media behavior of `go-whatsapp-web-multidevice` (GoWA) and provides an implementation guide for a new project that archives WhatsApp conversations and exports them to HTML or PDF.

> **Scope and version note:** This is an implementation reference based on the repository's current `main` API and the chat-storage/media code around the analyzed revision. GoWA is an unofficial WhatsApp Web client, not the WhatsApp Business Platform. WhatsApp protocol behavior, media URLs, and stored protobuf fields can change; treat stored media as best-effort archival input and test against the exact GoWA/whatsmeow version you deploy.

## 1. Executive summary

GoWA is a long-running Go server built on `whatsmeow`, Fiber, and SQLite. It receives WhatsApp events, converts them into normalized chat/message records, and exposes the records through a device-scoped REST API. Incoming media can be downloaded automatically or on demand. The repository does **not** implement a chat HTML/PDF export feature; it provides the storage, message query, and media-download primitives from which an exporter should be built.

The recommended architecture for a new exporter is:

1. Select a registered WhatsApp device.
2. Enumerate chats with `GET /chats`.
3. Page through one chat at a time with `GET /chat/{chat_jid}/messages`.
4. Identify media messages from `media_type`, download missing media through `GET /message/{message_id}/download`, and copy or reference the returned file.
5. Normalize every message into an immutable export model.
6. Render the model into self-contained HTML with escaped text and local media URLs.
7. Render HTML to PDF with a browser engine or a command-line renderer such as Chromium; do not rely on PDF generation to fetch expiring WhatsApp URLs.

For high-volume or offline exports, read the SQLite chat-storage database directly using a read-only connection, but keep the API/device identity rules and media-download path as the behavioral reference.

## 2. Repository map for data and media

| Concern | Primary location | What it establishes |
|---|---|---|
| Public chat API | `src/ui/rest/chat.go` | Chat listing, message paging, filtering, and device context |
| Public media API | `src/ui/rest/message.go` | `GET /message/:message_id/download` and safe public static URLs |
| Chat/message DTOs | `src/domains/chat/chat.go` | REST response fields used by an exporter |
| Storage entities and filters | `src/domains/chatstorage/chatstorage.go` | Database-level model and media metadata |
| Storage contract | `src/domains/chatstorage/interfaces.go` | Query, count, delete, edit, and device operations |
| SQLite implementation | `src/infrastructure/chatstorage/sqlite_repository.go` | Schema migrations and SQL repository |
| Device isolation wrapper | `src/infrastructure/whatsapp/chatstorage_wrapper.go` | Injects the current device into storage operations |
| Media reconstruction/download | `src/pkg/utils/whatsapp.go` | Media types, WhatsApp metadata, decryption download, filenames |
| Download use case | `src/usecase/message.go` | Validates, resolves, downloads, and stores media |
| Runtime paths/configuration | `src/config/settings.go` | `statics/media`, database URI, size limits, auto-download |
| OpenAPI reference | `docs/openapi.yaml` | Endpoint and response contracts |

The Go module is under `src/`, not the repository root. Runtime paths are relative to the process working directory. A local run from `src/` uses `src/storages` and `src/statics`; Docker mounts root-level `storages` and `statics` into `/app`.

## 3. Storage model

### 3.1 Device is part of the data identity

GoWA supports multiple WhatsApp accounts in one process. A device ID is therefore a required partition key for user-facing chat/message operations.

Important identities:

- **API device ID:** the registered slot selected by `X-Device-Id` or `device_id` query parameter.
- **WhatsApp storage device ID:** normally the logged-in client's bare/non-AD JID (`client.Store.ID.ToNonAD().String()`), used to partition chat storage.
- **Chat identity:** `(device_id, jid)`.
- **Message identity:** `(device_id, chat_jid, id)`.

Never merge rows by message ID alone in an exporter. The same message ID can be present in independent device stores, and the repository explicitly provides `GetMessageByIDChatAndDevice` for the full identity lookup.

For an export job, persist at least:

```text
export.device_id
export.chat_jid
export.message_id
```

Do not derive `device_id` from a display name or from an arbitrary phone number. Resolve it through the device API, then retain it in the export manifest.

### 3.2 Chat record

The storage `Chat` entity contains:

| Field | Meaning |
|---|---|
| `DeviceID` | Device partition |
| `JID` | WhatsApp conversation JID, for example user or group JID |
| `Name` | Resolved display/chat name |
| `LastMessageTime` | Last known message time |
| `EphemeralExpiration` | Disappearing-message setting in seconds |
| `Archived` | Whether the chat is archived |
| `CreatedAt`, `UpdatedAt` | Local storage timestamps |

The REST `ChatInfo` is a presentation form of this entity. `status@broadcast` is intentionally displayed as `Status` by the storage layer.

### 3.3 Message record

The storage `Message` entity is the canonical input for an exporter:

| Field | Export use |
|---|---|
| `ID` | Stable WhatsApp message ID; use as the export anchor |
| `ChatJID` | Conversation partition |
| `DeviceID` | Account/device partition |
| `Sender` | Sender JID; resolve a display label separately |
| `Content` | Text, caption, or normalized summary for structured messages |
| `Timestamp` | WhatsApp message time; use for chronological ordering |
| `IsFromMe` | Direction (`outgoing` vs `incoming`) |
| `MediaType` | Empty for text/structured-only messages; values include `image`, `video`, `video_note`, `audio`, `ptt`, `document`, `sticker`, and synthetic `call` |
| `CallMetadata` | JSON metadata when `MediaType == "call"` |
| `Filename` | Original/generated media filename |
| `URL` | WhatsApp media URL, potentially temporary |
| `DirectPath` | WhatsApp media direct path; preferred reconstruction input when present |
| `MediaKey` | Encrypted-media key required by whatsmeow to decrypt/download |
| `FileSHA256` | Plain media hash/check data |
| `FileEncSHA256` | Encrypted media hash/check data |
| `FileLength` | Expected media size |
| `ReferralMetadata` | JSON metadata for external-ad/click-to-WhatsApp attribution |
| `Reactions` | Related reaction records loaded for REST responses |
| `CreatedAt`, `UpdatedAt` | Local persistence/update timestamps |

The REST message representation intentionally omits sensitive media-key/hash fields. It exposes `id`, `chat_jid`, sender information, content, timestamp, direction, media type, filename, URL, file length, call metadata, reactions, and local timestamps.

### 3.4 Edits, reactions, polls, and calls

A faithful export should not treat a message row as only text:

- **Edits:** `message_edits` is append-only history. The current message content is updated, while each edit retains original message ID, edit event ID, editor, previous content, new content, and edit time. Export both the final value and an optional edit history section.
- **Reactions:** reactions are associated with messages and returned in the REST `Reactions` field. Export the emoji, reactor, and reaction time where available.
- **Polls:** poll definitions and options are stored separately because vote hashes must be resolved after restart/history synchronization. Preserve the question, stable option names, option hashes only if needed for audit, and resolved votes if available from the API/event model.
- **Calls:** incoming call offers are persisted as synthetic messages with `media_type = "call"` and JSON `call_metadata`. Render them as event cards, not media attachments.
- **Contacts and locations:** message `content` may be a normalized summary, such as `Contact: ...` or a Google Maps link. Do not assume every non-empty content value is ordinary user text.
- **Replies/quotes:** the current message storage entity does not expose a complete quoted-message object. If quote context is needed, capture it from webhook/event payloads or add a versioned storage field rather than trying to infer it from plain content.

## 4. Querying chat data through REST

All device-scoped routes require either:

```http
X-Device-Id: <registered-device-id>
```

or:

```text
device_id=<registered-device-id>
```

The middleware resolves the header first, then the query parameter, and otherwise falls back only when a default/only device can be resolved. A missing or invalid device must be treated as an error, not as an invitation to query all devices.

### 4.1 Enumerate chats

```http
GET /chats?limit=100&offset=0&has_media=false&archived=false
X-Device-Id: my-device
```

Supported filters:

- `limit`, `offset`: pagination.
- `search`: chat-name search.
- `has_media=true`: chats with stored media.
- `archived=true|false`: explicit archive filter.

The response includes `data` and `pagination.total`. Use the total or continue until a page is empty. Do not assume the default page size is sufficient for a complete archive.

### 4.2 Enumerate messages

The chat JID must be URL path encoded. In particular, encode `@` as `%40` when required by the client/router.

```http
GET /chat/120363012345678901%40g.us/messages?limit=100&offset=0
X-Device-Id: my-device
```

Supported filters:

- `limit`, `offset`: page through the chat.
- `start_time`, `end_time`: time-window export filters; use the format accepted by the running API version.
- `media_only=true`: media-focused pass.
- `is_from_me=true|false`: direction filter.
- `search`: database-level message text search.

The response contains `data`, `pagination`, and `chat_info`. For deterministic exports, sort by message timestamp in the exporter and use a stable tie-breaker such as message ID. The API/database ordering should be verified against the deployed version before relying on it for incremental checkpoints.

### 4.3 Incremental export strategy

A robust exporter should maintain a manifest per `(device_id, chat_jid)`:

```json
{
  "device_id": "my-device",
  "chat_jid": "120363012345678901@g.us",
  "last_timestamp": "2026-09-18T12:00:00Z",
  "last_message_id": "ABC123",
  "media": {"downloaded": 42, "missing": 1},
  "source_revision": "gowa-v9.x"
}
```

Use an overlap window (for example, re-read the last few minutes or last page) and deduplicate by `(device_id, chat_jid, message_id)`. This protects against late history synchronization and messages whose local rows are updated after the first export. Store a content hash of the normalized export record if audit-grade change detection is required.

## 5. Media lifecycle

### 5.1 Automatic versus on-demand download

`WHATSAPP_AUTO_DOWNLOAD_MEDIA` / `--auto-download-media` defaults to `true`. Automatic downloading is event-driven and writes media under the configured media path. It is convenient but can consume substantial disk space and may not cover older messages that arrived before a configuration change.

For an exporter, use an explicit two-phase process:

1. Export all message metadata first.
2. Download media on demand for rows where the local asset is missing.

This makes text-only exports possible even when media is unavailable and allows retry/reporting for expired media.

### 5.2 Stored media metadata is not the media file

A message row stores encrypted WhatsApp references and integrity metadata, not necessarily a local file. The `URL` can expire. `DirectPath` is preferred; `ResolveMediaDirectPath` derives a request path from legacy full URLs when `DirectPath` is empty. `MediaKey`, both SHA-256 fields, and `FileLength` are needed to reconstruct the whatsmeow downloadable message.

The download path is:

1. Load the message row.
2. Resolve `DirectPath` from `DirectPath` or legacy `URL`.
3. Rebuild a typed whatsmeow media message with `BuildDownloadableMessage`.
4. Call the logged-in client download operation.
5. Validate the downloaded byte size against `WhatsappSettingMaxDownloadSize`.
6. Write a file with mode `0600`.

A new project should not attempt to decrypt WhatsApp media by simply issuing an HTTP GET against `URL`. Use the GoWA endpoint or the same whatsmeow client/session.

### 5.3 Supported media types

The repository extracts and reconstructs these types:

| `media_type` | Typical output | HTML/PDF treatment |
|---|---|---|
| `image` | JPEG or source-compatible image | Inline thumbnail/full image when available |
| `video` | MP4 or downloaded source | Poster/filename plus optional video link; PDF usually uses poster only |
| `video_note` | Circular video | Poster/filename; preserve original as attachment |
| `audio` | Audio file | Player in HTML; filename/duration placeholder in PDF |
| `ptt` | Voice note, typically OGG/Opus | Player in HTML; transcript or filename in PDF |
| `document` | Original document filename/extension | Download link in HTML; attachment reference in PDF |
| `sticker` | WebP | Inline image in HTML; rasterize or link in PDF if renderer lacks WebP |
| `call` | No media file | Event card from `call_metadata` |

The media type may differ from the MIME type. Preserve both the logical WhatsApp type and the detected file MIME type in the export manifest.

### 5.4 Media endpoint

```http
GET /message/{message_id}/download?phone=<chat-or-recipient>
X-Device-Id: my-device
```

The endpoint validates the message, verifies that it belongs to the supplied chat, downloads it, and returns a normal GoWA response containing:

```json
{
  "status": 200,
  "code": "SUCCESS",
  "results": {
    "message_id": "ABC123",
    "status": "Media downloaded successfully to statics/media/628123456789/2026-09-18/1700000000-uuid.jpg",
    "media_type": "image",
    "filename": "1700000000-uuid.jpg",
    "file_path": "statics/media/628123456789/2026-09-18/1700000000-uuid.jpg",
    "file_url": "/statics/media/628123456789/2026-09-18/1700000000-uuid.jpg",
    "file_size": 123456
  }
}
```

When `file_url` is not already set, the REST handler builds it only for paths safely relative to `statics/`. It URL-escapes path components and rejects traversal outside that directory. Prefer copying the file into an exporter-owned immutable archive directory instead of depending on a mutable runtime path.

### 5.5 Media directory layout

The built-in on-demand downloader uses:

```text
<media-root>/<phone-number>/<YYYY-MM-DD>/<generated-file>
```

The phone number is extracted from the chat JID. This is useful operationally, but it is not a sufficient archive key for groups, `@lid` identities, newsletters, or multiple devices. An exporter should use a collision-resistant layout such as:

```text
archive/
  <device-id>/
    chats/<sha256(chat-jid)>/
      media/<message-id>-<safe-filename>
```

Store the original JID and original filename in a manifest. Never use an untrusted WhatsApp filename or JID directly as a filesystem path; sanitize it and bound its length.

## 6. Rendering a durable export

### 6.1 Canonical export model

Normalize API/storage data before rendering. A useful model is:

```json
{
  "schema_version": 1,
  "device_id": "my-device",
  "chat": {
    "jid": "120363012345678901@g.us",
    "name": "Project Group",
    "archived": false
  },
  "messages": [
    {
      "id": "ABC123",
      "timestamp": "2026-09-18T12:34:56Z",
      "sender_jid": "628123456789@s.whatsapp.net",
      "sender_name": "Asha",
      "direction": "incoming",
      "content": "See attached photo",
      "media": {
        "type": "image",
        "filename": "image_20260918_123400.jpg",
        "relative_path": "media/ABC123-image.jpg",
        "mime_type": "image/jpeg",
        "size": 123456,
        "sha256": "...",
        "available": true
      },
      "reactions": [],
      "edited": false,
      "source": "gowa-rest"
    }
  ]
}
```

Keep the raw API response or a redacted raw snapshot alongside the normalized model if the export has evidentiary or compliance requirements. Do not store `MediaKey` in ordinary HTML/PDF output.

### 6.2 HTML requirements

Generate a self-contained or portable HTML package:

```text
export-2026-09-18/
  index.html
  manifest.json
  media/
  thumbnails/
  attachments/
```

Recommended rules:

- Escape message text as HTML; never concatenate untrusted content into markup.
- Use `textContent`/template escaping if rendering client-side.
- Use local relative media paths, not expiring WhatsApp URLs.
- Render sender, timestamp, direction, message ID, and media status.
- Add a visible “media unavailable” marker with the reason: not downloaded, expired, permission failure, unsupported type, or size limit.
- Render captions as message text next to the media.
- Preserve line breaks and Unicode/emoji with UTF-8.
- Use CSS `break-inside: avoid` for message cards where possible.
- Include a generated-at timestamp, source device ID, and export schema version.

For large archives, split by chat and date and create an index rather than producing a single multi-gigabyte HTML file.

### 6.3 PDF requirements

Treat PDF as a presentation derivative, not the system of record:

- Generate from the archived HTML and local media, never from live WhatsApp URLs.
- Use a renderer with explicit local-file permissions and a controlled timeout.
- Use a print stylesheet; hide audio/video controls and show a stable attachment card.
- Include page headers/footers with chat name, date range, page number, and device identifier where appropriate.
- Rasterize WebP/stickers to PNG/JPEG when the PDF engine cannot embed WebP.
- Keep the original HTML, manifest, and media directory so the PDF can be regenerated.
- Record which media was omitted and why; a successful PDF process must not silently turn missing media into blank space.

## 7. Recommended exporter workflow

### REST-first workflow

```text
resolve device
  -> list chats
  -> for each chat, page messages
  -> normalize records
  -> for each media row, locate local asset
       -> if absent, call /message/:id/download
       -> verify file exists and hash it
  -> write manifest
  -> render HTML
  -> render PDF from local HTML
```

Pseudo-code:

```text
for chat in paginate(GET /chats, device_id):
    messages = paginate(GET /chat/{chat.jid}/messages, device_id)
    for message in messages:
        record = normalize(message, chat, device_id)
        if message.media_type != "" and not local_asset(record):
            result = GET /message/{message.id}/download?phone={chat.jid}
            if result succeeds:
                copy_to_archive(result.file_path, record)
            else:
                record.media.available = false
                record.media.error = classify_download_error(result)
        append(record)
    write_chat_html(chat, records)
write_manifest()
render_pdf_from_local_html()
```

### Direct SQLite workflow

Direct SQLite access is appropriate for a controlled offline tool or a high-volume exporter that runs beside GoWA. The configured default chat DB is `file:storages/chatstorage.db`; the WhatsApp session DB is separate (`file:storages/whatsapp.db`). Do not treat the WhatsApp session database as the chat archive.

Use a read-only connection and query the chat-storage schema through the documented entity shape. Include `device_id` in every query. Direct SQLite gives efficient access to historical rows, but it cannot download media by itself: media download still requires an authenticated, connected whatsmeow client or GoWA's media endpoint. If the WhatsApp URL has expired and no active session can re-fetch it, mark the asset unavailable.

Do not modify GoWA's database from the exporter. If adding exporter-specific state, use a separate database or append-only manifest. GoWA migrations are inline and append-only; changing its schema from an external tool risks breaking upgrades.

## 8. Reliability, consistency, and deletion behavior

- **Late synchronization:** WhatsApp history can arrive after the initial query. Use overlapping incremental windows and deduplication.
- **Edits:** re-export or update the message record when `UpdatedAt` changes; retain edit history where required.
- **Deletes/revokes:** local message deletion can remove the stored row. Export jobs should checkpoint frequently and should not claim completeness without a deletion/event policy. Webhook consumers can preserve tombstones before the row disappears.
- **Disappearing messages:** `EphemeralExpiration` and message context can indicate disappearing content. An archival project must explicitly define whether it is allowed to retain such content.
- **Media URL expiry:** download as soon as practical, but always support missing-media output.
- **Multiple devices:** keep separate export roots/manifests unless the product explicitly supports cross-device merging.
- **Process restarts:** runtime storage is persisted only when `storages` and `statics` are mounted/preserved. Docker Compose mounts both directories for this reason.
- **Disk usage:** automatic media download has a 500 MB default download limit per file, while upload limits are 20 MB images, 50 MB files, and 100 MB videos. Configure and monitor limits before bulk archival.

## 9. Security and privacy checklist

1. Protect REST and media routes with Basic Auth or a trusted reverse proxy; never expose `/statics` publicly without considering the sensitivity of files.
2. Send the correct `X-Device-Id` on every request and verify that the response belongs to that device.
3. Treat JIDs, message content, filenames, captions, contact cards, location links, and document contents as untrusted data.
4. Escape HTML and sanitize filesystem paths.
5. Do not put `MediaKey`, encrypted hashes, session databases, or API credentials in an export.
6. Keep exported archives encrypted at rest when they contain personal or regulated data.
7. Avoid putting Basic Auth credentials in URLs. The repository documents a WebSocket query-string workaround for browsers, but that exposes credentials in logs and should only be used over TLS with care.
8. Apply retention rules to both the normalized export and downloaded media.
9. Log download failures without logging raw message content, media keys, tokens, or secrets.
10. Validate media MIME type and size before handing a file to an image/PDF/document parser to reduce parser and decompression-bomb risk.

## 10. Known limitations and design decisions

- GoWA exposes chat/message retrieval and media download, not a stable public export format. Define and version your own export schema.
- The REST message response does not include all storage metadata. Use the API for supported behavior; use a controlled direct-storage reader only when the extra fields are necessary.
- Stored `URL` values are transport metadata, not durable public URLs.
- Generated media names include a current timestamp and UUID; do not use them as the message's stable identity.
- The default media directory is organized by phone number/date, which can collide or be ambiguous across groups, LIDs, newsletters, and devices.
- Chatwoot direct Postgres import is unrelated to HTML/PDF export. It is for historical Chatwoot synchronization and must not be confused with GoWA's live WhatsApp chat storage or live REST media forwarding.
- WhatsApp and whatsmeow may add message wrappers or new message types. Preserve unknown payloads as an “unsupported/unknown” record rather than dropping the message.

## 11. Validation checklist for a new project

Before calling an export complete, verify:

- [ ] The selected device ID is recorded and every query is device-scoped.
- [ ] Every chat page and message page was consumed; pagination totals are recorded.
- [ ] Message IDs are deduplicated by device and chat.
- [ ] Messages are ordered by WhatsApp timestamp with a deterministic tie-breaker.
- [ ] Incoming, outgoing, group, user, newsletter, status, and `@lid` cases are tested.
- [ ] Text, captions, contacts, locations, polls, reactions, edits, calls, and disappearing-message metadata have explicit rendering rules.
- [ ] Media is copied to an archive-owned path and hashed.
- [ ] Missing/expired/unsupported media is visible in both the manifest and HTML.
- [ ] HTML escaping, filename sanitization, and local-file renderer permissions are tested.
- [ ] PDF output is regenerated only from the archived HTML/media package.
- [ ] The source database/session files are backed up separately from the export.
- [ ] Retention, access control, encryption, and deletion rules are documented.

## 12. Source references

- [README and API overview](../readme.md)
- [OpenAPI specification](docs/openapi.yaml)
- [Chat REST adapter](src/ui/rest/chat.go)
- [Message REST adapter and media URL handling](src/ui/rest/message.go)
- [Chat/message REST DTOs](src/domains/chat/chat.go)
- [Storage entities and media metadata](src/domains/chatstorage/chatstorage.go)
- [Storage contract](src/domains/chatstorage/interfaces.go)
- [Media extraction and download reconstruction](src/pkg/utils/whatsapp.go)
- [Message download use case](src/usecase/message.go)
- [Runtime paths and size limits](src/config/settings.go)
- [Chat-storage implementation guidance](src/infrastructure/chatstorage/AGENTS.md)
