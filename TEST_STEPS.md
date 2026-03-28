# Local testing: APK download with HTTP Basic Auth

This document describes how to build **gohttpserver** from source, run it with authentication, and verify that **`.apk`** responses include `Content-Disposition: attachment` so Android browsers can download reliably.

**Repository:** https://github.com/codeskyblue/gohttpserver  
**Issue:** https://github.com/codeskyblue/gohttpserver/issues/136  

---

## 1. Build

From the repository root (the folder that contains `main.go`):

```bash
cd /path/to/gohttpserver
go test ./...
go build -o /tmp/gohttpserver-test .
```

---

## 2. Prepare a test APK

You need any valid `.apk` file (e.g. a debug build). Copy it into a dedicated directory used as `--root`, for example:

```bash
mkdir -p /tmp/ghs-root
cp /path/to/your.apk /tmp/ghs-root/test.apk
```

---

## 3. Scenario A — HTTP Basic Auth enabled

### Start the server

```bash
/tmp/gohttpserver-test \
  --port 8080 \
  --root /tmp/ghs-root \
  --auth-type http \
  --auth-http admin:yourpassword \
  --upload
```

Use the username and password you passed to `--auth-http` (format `user:pass`).

### Verify headers with curl (no Android required)

```bash
curl -sI -u admin:yourpassword "http://127.0.0.1:8080/test.apk"
```

**Expected:** a response line similar to:

```http
Content-Disposition: attachment; filename="test.apk"
```

Also confirm `200 OK` and a non-zero `Content-Length` (or chunked body) when using `GET` without `-I`.

### Android device (same Wi‑Fi as the PC)

1. Find the PC’s LAN IP, e.g. `ip -4 addr show` or `hostname -I` (example: `192.168.1.50`).
2. On the phone, open: `http://192.168.1.50:8080/test.apk`
3. Enter Basic Auth when prompted.
4. **Expected:** the browser starts a download or opens the package installer; no “invalid link” / failed download attributable to missing attachment disposition.

### Optional: explicit download query (should still work)

```
http://192.168.1.50:8080/test.apk?download=true
```

This path already forced attachment before the APK-specific fix; both should behave consistently.

---

## 4. Scenario B — No auth (baseline)

```bash
/tmp/gohttpserver-test --port 8081 --root /tmp/ghs-root --upload
```

Open `http://127.0.0.1:8081/test.apk` on Android or desktop. Download should succeed; use this to compare behavior if auth scenarios fail.

---

## 5. Regression checks (other file types)

With **auth enabled** as in scenario A:

| File | Upload / place in root | Expected |
|------|-------------------------|----------|
| `photo.jpg` | yes | Image **displays** in browser (not forced download). |
| `notes.txt` | yes | Text **displays** inline. |
| `test.apk` | yes | **Download** / install flow; `Content-Disposition: attachment`. |

---

## 6. Optional: QR code from the web UI

If you use the built-in listing UI:

1. Open `http://<host>:8080/` in a desktop browser and authenticate.
2. Use the QR action next to `test.apk` if your theme exposes it.
3. Scan with the phone; expect the same successful download as a direct URL.

---

## 7. If something fails

Collect:

- Exact browser message and OEM (e.g. Huawei Browser, Chrome).
- Android version.
- Response headers from:  
  `curl -sI -u user:pass "http://<host>:<port>/test.apk"`
- Server log line for the `GET` (gohttpserver logs the path).

---

## 8. Other auth modes

The reported issue targets **HTTP Basic Auth** (`--auth-type http`). Other modes (`openid`, etc.) use different flows; if you test them, document the command line from `--help` and repeat the APK header check with `curl` where applicable.
