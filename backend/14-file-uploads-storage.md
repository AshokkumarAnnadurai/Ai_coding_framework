# Backend Standards: File Uploads & Storage

Rules for validating, storing, and serving user-uploaded files safely.

## 1. Upload Validation

*   **Allow-List File Types:** Validate uploaded files against an explicit allow-list of accepted MIME types — checked from the file's actual content signature (magic bytes), not just the client-supplied `Content-Type` header or filename extension, both of which are trivially spoofable.
*   **Enforce Size Limits Server-Side:** Set explicit size limits on Multer (or equivalent) at the framework level (`limits: { fileSize }`) — never rely on client-side size checks alone.
*   **Reject Unexpected Fields:** Only accept the specific field name(s) the endpoint expects; reject requests with unexpected additional file fields.

```typescript
@Post('avatar')
@UseInterceptors(
  FileInterceptor('file', {
    limits: { fileSize: 5 * 1024 * 1024 },
    fileFilter: (req, file, callback) => {
      const allowed = ['image/png', 'image/jpeg', 'image/webp'];
      callback(null, allowed.includes(file.mimetype));
    },
  }),
)
async uploadAvatar(@UploadedFile() file: Express.Multer.File): Promise<UploadResponseDto> {
  return this.uploadService.storeAvatar(file);
}
```

---

## 2. Never Trust Client-Supplied Filenames

*   **Regenerate Filenames Server-Side:** Store uploaded files under a server-generated identifier (UUID) plus a validated extension — never use the client-provided filename directly for the stored path.
*   **Prevent Path Traversal:** Never interpolate any part of a client-supplied filename or path into a filesystem path. Combined with server-generated names above, this closes off traversal attacks (`../../etc/passwd`-style payloads) entirely.

---

## 3. Storage Location

*   **Prefer Object Storage Over Local Disk:** Use S3 (or an equivalent object store) for anything beyond ephemeral/local-dev use. Local disk storage doesn't survive container restarts and doesn't scale across multiple instances.
*   **Keep Storage Access Behind a Service:** Route all storage reads/writes through a dedicated storage service/adapter (see [11-external-integrations-resilience.md](11-external-integrations-resilience.md#1-adapter-pattern-for-third-party-services)) so the storage backend can be swapped without touching business logic.

---

## 4. Malware Scanning

*   **Scan Before Serving:** User-uploaded files that will be downloaded by other users (not just the uploader) should be scanned for malware before being made available, particularly documents and executables.
*   **Quarantine on Failure:** A file that fails a scan must not be silently discarded without record — flag it, retain it in a quarantined location for review, and do not expose it via any download path.

---

## 5. Access Control on Stored Files

*   **No Public, Guessable URLs for Sensitive Files:** Files containing PHI/PII or other sensitive content must never be stored at a publicly-readable, guessable URL. Use short-lived signed/pre-signed URLs generated per-request after an authorization check.
*   **Authorize Every Download:** A download endpoint must re-verify the requesting user's authorization to access that specific file — do not assume possession of a URL implies authorization.
