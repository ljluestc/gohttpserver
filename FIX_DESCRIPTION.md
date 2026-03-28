# APK + HTTP Basic Auth — fix (local reference)

This file is a short pointer to the full write-up. **Authoritative English docs:**

| Document | Purpose |
|----------|---------|
| [PR_DESCRIPTION.md](./PR_DESCRIPTION.md) | Problem, root cause, code change summary, upstream links, maintainer checklist |
| [TEST_STEPS.md](./TEST_STEPS.md) | Build, run, `curl` and Android verification |

**Upstream**

- Repo: https://github.com/codeskyblue/gohttpserver  
- Issue: https://github.com/codeskyblue/gohttpserver/issues/136  

**Code**

- File: `httpstaticserver.go`, function `hIndex`  
- Behavior: for paths with extension `.apk`, set `Content-Disposition: attachment` before `http.ServeFile` (same idea as `?download=true`).
