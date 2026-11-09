[make.fun API](../README.md) / **Errors**

> Generated from [make.fun/developers](https://make.fun/developers), docs version 1.0.1. An edit here is overwritten by the next update: changes belong upstream, in make.fun itself.

# Errors

Every error is one JSON shape: `{"success": false, "error": [{"message": "...", "path": [...]}], "data": ...}`. `data` carries the details, as `data.code` on a refused key.

A refused entry or claim, and a 429, is an RFC 9457 problem (`application/problem+json`) that carries the same `success` and `error` members too: `type`, `title`, `status`, `detail`, and for an entry `checks` with each failed check's `fix`. `GET /problems/<id>` explains a type (`https://make.fun/problems/<check>` for an entry's checks).

---

Next: [Endpoint reference](endpoints.md)
