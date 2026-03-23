# Storage — local disk or S3-compatible object storage

## Investigation Summary

Paperclip provides a pluggable storage layer for uploaded files (issue attachments, company logos, images). Two providers are implemented: local disk (default, zero-config) and S3-compatible (for production/cloud). The storage layer is a clean abstraction (~450 lines total) with company-scoped object keys, SHA-256 integrity hashing, and path traversal protection. It sits between the asset management API and the underlying filesystem/object store.

---

## 1. Architecture

```
Routes (upload/download) → StorageService → StorageProvider → Filesystem / S3
                              ↓
                          assets table (DB metadata)
```

- **StorageProvider** — low-level put/get/head/delete on object keys
- **StorageService** — high-level file operations with company scoping, key generation, hashing
- **Provider Registry** — creates the right provider from config

---

## 2. Storage Providers

### 2.1 Local Disk (Default)

**Location:** `~/.paperclip/instances/default/data/storage/`

**Implementation (~90 lines):**
- `putObject` — write to temp file, then atomic rename
- `getObject` — `createReadStream()` + stat for size/mtime
- `headObject` — `stat()` check
- `deleteObject` — `unlink()`, idempotent (catches errors)

**Path safety:**
- `normalizeObjectKey()` — rejects empty, absolute, `.`, `..` segments
- `resolveWithin()` — resolves path and verifies it's within the base directory (prevents path traversal)

### 2.2 S3-Compatible

**Supported services:** AWS S3, MinIO, Cloudflare R2, any S3-compatible API.

**Configuration:**
```typescript
{
  bucket: string,          // Required
  region: string,          // Required
  endpoint?: string,       // Custom endpoint (MinIO, R2)
  prefix?: string,         // Object key prefix
  forcePathStyle?: boolean // For MinIO-style endpoints
}
```

**Implementation (~130 lines):**
- Uses `@aws-sdk/client-s3` (`S3Client`, `PutObjectCommand`, `GetObjectCommand`, `HeadObjectCommand`, `DeleteObjectCommand`)
- Handles S3 body stream conversion (Web ReadableStream → Node Readable)
- Handles `NoSuchKey`/`NotFound` errors gracefully

---

## 3. Storage Service

The service layer adds company scoping, key generation, and integrity:

### 3.1 Object Key Format

```
{companyId}/{namespace}/{year}/{month}/{day}/{uuid}-{sanitized-filename}{ext}
```

Example: `abc123/attachments/2026/03/23/d4e5f6a7-report.pdf`

- **Company prefix** — enforced on every read/write/delete (prevents cross-company access)
- **Namespace** — caller-provided category (e.g., `attachments`, `logos`)
- **Date partitioning** — UTC year/month/day directories
- **UUID prefix** — prevents filename collisions
- **Sanitized filename** — alphanumeric + `._-`, max 120 chars per segment

### 3.2 File Upload (`putFile`)

1. Validate input (companyId, namespace, contentType, non-empty Buffer)
2. Generate object key
3. Delegate to provider's `putObject()`
4. Return: `{ provider, objectKey, contentType, byteSize, sha256, originalFilename }`

### 3.3 Access Control

`ensureCompanyPrefix(companyId, objectKey)`:
- Object key must start with `{companyId}/`
- Rejects keys containing `..`
- Called on every `getObject`, `headObject`, `deleteObject`

---

## 4. Asset Management (DB Layer)

Stored files are tracked in the `assets` table:

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID | PK |
| `companyId` | UUID | FK → companies |
| `provider` | TEXT | `"local_disk"` or `"s3"` |
| `objectKey` | TEXT | Storage path/key |
| `contentType` | TEXT | MIME type |
| `byteSize` | INT | File size |
| `sha256` | TEXT | Integrity hash |
| `originalFilename` | TEXT | Original upload filename |
| `createdByAgentId` | UUID | FK → agents |
| `createdByUserId` | TEXT | — |

**Serving:** `GET /api/assets/{assetId}/content` streams the file from storage with the stored content type.

---

## 5. Configuration

### 5.1 Via Config File

```json
// ~/.paperclip/instances/default/config.json
{
  "storageProvider": "local_disk",
  "storageLocalDiskBaseDir": "~/.paperclip/instances/default/data/storage"
}
// or
{
  "storageProvider": "s3",
  "storageS3Bucket": "my-bucket",
  "storageS3Region": "us-east-1",
  "storageS3Endpoint": "https://s3.us-east-1.amazonaws.com",
  "storageS3Prefix": "paperclip",
  "storageS3ForcePathStyle": false
}
```

### 5.2 Via CLI

```sh
pnpm paperclipai configure --section storage
```

---

## 6. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Provider interface abstraction | Swap between local disk and S3 without changing callers |
| Company-prefixed object keys | Prevents cross-company access at the storage layer |
| Atomic write (temp file + rename) for local disk | Prevents partial file reads on crash |
| SHA-256 hash stored per asset | Integrity verification, deduplication potential |
| UUID-prefixed filenames | Prevents collisions without coordination |
| Date-partitioned directories | Prevents single-directory bloat, makes cleanup easier |
| Path traversal protection | Defense-in-depth — validate at both service and provider level |
| Idempotent delete | deleteObject succeeds even if file doesn't exist |

---

## 7. Observations for Colmena

1. **The storage abstraction is clean and worth adopting as-is** — ~450 lines total for a pluggable storage layer with two providers.

2. **For Colmena MVP, local disk only.** Add S3 when deploying to cloud or needing shared storage across instances.

3. **The company-prefix enforcement** is essential for multi-tenancy — every storage operation validates the company owns the object.

4. **The atomic write pattern** (temp file + rename) is a small but important detail for reliability.

5. **The asset metadata table** decouples storage from the rest of the system — routes serve files by asset ID, not by storage key. This makes storage migration transparent.

6. **Date-partitioned keys** (`2026/03/23/`) are good practice — prevents single-directory issues and makes it easy to archive/clean old files.

7. **Missing features for production:** no upload size limits enforced at storage layer (handled at route level via multer), no CDN integration, no signed URLs for direct client uploads. These could be added later.

8. **The S3 provider** handles the common pitfalls: `forcePathStyle` for MinIO, web stream → Node stream conversion, `NoSuchKey` error mapping. Production-ready.
