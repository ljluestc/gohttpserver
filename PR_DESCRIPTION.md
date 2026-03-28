# Fix: APK download fails on Android when HTTP Basic Auth is enabled

## Summary

When the server runs with `--auth-type http`, some Android browsers (notably stock browsers on Huawei and other OEMs) fail to download `.apk` files, often showing errors such as “link invalid” or “download failed.” Other file types (e.g. `.txt`) still open or download as expected. This change sets `Content-Disposition: attachment` for all APK responses so the browser treats the file as a download, matching the behavior already available via `?download=true` and avoiding fragile inline/package-install handling behind authenticated sessions.

## Upstream references

| Resource | URL |
|----------|-----|
| Repository | https://github.com/codeskyblue/gohttpserver |
| Related issue | https://github.com/codeskyblue/gohttpserver/issues/136 |

## Problem

### Steps to reproduce

1. Build and run with HTTP Basic Auth, for example:
   ```bash
   go build -o gohttpserver .
   ./gohttpserver --port 8080 --auth-type http --auth-http admin:secret --upload
   ```
2. Place or upload an APK under the document root (e.g. `app-release.apk`).
3. On an Android device on the same network, open the direct file URL in the browser (after completing Basic Auth).
4. **Observed (before fix):** Download fails or the browser reports an invalid link; behavior varies by OEM.
5. **Control:** With auth disabled, the same APK URL often works. With auth enabled, `?download=true` may work because it already sets `Content-Disposition: attachment`.

### Expected behavior

- With HTTP auth enabled, APKs should download (or open the system installer) reliably, consistent with using `?download=true`.

## Root cause (analysis)

Android WebView / stock browsers apply stricter handling for `application/vnd.android.package-archive` responses. Without an explicit `Content-Disposition: attachment` (and a stable filename), some builds refuse to complete the download when the request is authenticated (extra security prompts and MIME sniffing). Forcing attachment aligns APK delivery with explicit download semantics and reduces OEM-specific failures.

## Solution

In `httpstaticserver.go`, function `hIndex`, before `http.ServeFile`:

- If the request path has extension `.apk`, set:
  - `Content-Disposition: attachment; filename="<quoted-basename>"`
- Existing logic for `?download=true` is unchanged and still sets the same header for any file type.

Relevant code path: [`httpstaticserver.go`](https://github.com/codeskyblue/gohttpserver/blob/master/httpstaticserver.go) (search for `Fix #136` or `.apk`).

## Impact

- **Scoped to:** raw GET of files ending in `.apk` (same branch as `ServeFile`).
- **Unchanged:** directory listings, JSON APIs, upload/delete, IPA plist routes, other extensions.
- **Compatibility:** Same header pattern as the existing `download=true` branch; uses `strconv.Quote` for safe filename quoting.

## How to verify locally

See **[TEST_STEPS.md](./TEST_STEPS.md)** for build commands, auth/no-auth scenarios, Android checks, and optional `curl` header inspection.

### Quick smoke test (desktop)

```bash
go build -o /tmp/gohttpserver .
/tmp/gohttpserver --port 8080 --auth-type http --auth-http admin:secret --root ./testdata &
# Place a small test.apk under testdata or use -root pointing to a folder containing an APK
curl -sI -u admin:secret "http://127.0.0.1:8080/your.apk" | grep -i content-disposition
# Expect: Content-Disposition: attachment; filename="your.apk"
```

Stop the server when finished.

## Checklist for maintainers

- [ ] Reproduce on Android + HTTP auth before/after (see TEST_STEPS.md).
- [ ] Confirm non-APK files (images, text) still inline or behave as before.
- [ ] Close or link https://github.com/codeskyblue/gohttpserver/issues/136 when merged.
