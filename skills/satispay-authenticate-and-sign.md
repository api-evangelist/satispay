---
name: satispay-authenticate-and-sign
description: >-
  Pair an RSA key with a Satispay shop to get a KeyId, then sign every subsequent request using the
  Signing HTTP Messages (Cavage draft-10) scheme Satispay requires. There is no OAuth and no static
  API key.
api: Satispay GBusiness API
base_url: https://authservices.satispay.com/g_business/v1
operations:
- keyid
- testinput
generated: '2026-08-26'
method: generated
source: >-
  openapi/satispay-gbusiness-api.json, openapi/satispay-sandbox.json,
  https://developers.satispay.com/reference/introduction,
  https://developers.satispay.com/reference/compose-the-authentication-header,
  https://developers.satispay.com/docs/credentials
---

# Authenticate against the Satispay GBusiness API

Satispay does **not** use OAuth. There is no token endpoint, no scopes, and no static bearer key. Every
request carries an RSA signature computed over the request itself, following the
[Signing HTTP Messages](https://tools.ietf.org/html/draft-cavage-http-signatures-10) draft-10.

> Note the draft. It is not RFC 9421. An off-the-shelf RFC 9421 HTTP Message Signatures library will not
> interoperate without configuration.

## Pick the environment first

| | base URL |
|---|---|
| production | `https://authservices.satispay.com` |
| sandbox | `https://staging.authservices.satispay.com` |

TLS 1.2 or higher only. Credentials are **not** interchangeable and nothing in a `KeyId` identifies its
environment — a staging KeyId sent to production returns HTTP 404, code 41. If you get an unexplained
41, check the host before you check anything else.

## One-time setup

### 1. Generate an RSA key pair

Keep the private key secret. There is no rotate and no revoke operation in the API — if it leaks, you go
through the Dashboard and Satispay support.

### 2. Get the activation code

- **Sandbox**: request an account at <https://satispay-sandbox.paperform.co/>. A human provisions it and
  emails you the staging app plus the activation code. It is not self-service.
- **Production**: sign up at <https://dashboard.satispay.com>, get the business profile verified, create
  the shop, then generate the shop's activation code in the Dashboard.

### 3. Exchange it for a KeyId

`POST /g_business/v1/authentication_keys` (`keyid`), unsigned:

```json
{ "public_key": "-----BEGIN PUBLIC KEY-----\n...", "token": "<activation code>" }
```

Response: `{ "key_id": "..." }`. **Store it with the key pair.**

Activation codes are single use. Re-sending a paired one returns HTTP 403, code 45. A malformed
`public_key` returns HTTP 400, code 34.

You may send optional `x-satispay-*` device headers here (`x-satispay-devicetype` accepts `SMARTPHONE`,
`TABLET`, `CASH REGISTER`, `POS`, `PC`, `ECOMMERCE_PLUGIN`).

## Per-request signing

For every call:

1. **Digest** — `Digest: SHA-256=<base64(sha256(raw body))>`. Compute it over the exact bytes you send.
   For a GET with no body, digest the empty string.
2. **Message** — build the signing string from `(request-target)`, `host`, `date`, `digest`, in that
   order, each as `name: value` on its own line.
3. **Signature** — sign the message with the RSA private key, `rsa-sha256`, base64 the result.
4. **Authorization header**:

```
Authorization: Signature keyId="<KeyId>", algorithm="rsa-sha256",
  headers="(request-target) host date digest", signature="<base64>"
```

Also send `Accept: application/json` on everything, and `Content-Type: application/json` on POST and PUT.

## Verify before you build

`POST https://staging.authservices.satispay.com/wally-services/protocol/tests/signature` (`testinput`)
exists purely to tell you whether your signature verifies. It accepts GET, POST, PUT, DELETE and PATCH.
Get a green here before touching a payment endpoint.

## Failure modes

| code | HTTP | meaning |
|---|---|---|
| 34 | 400 | `public_key` invalid — check the PEM formatting |
| 34 | 401 | signature verified but merchant unknown or not entitled |
| 45 | 403 | activation code already paired |
| 41 | 404 | token not found — almost always the wrong environment |

## Firewall

Allow port 443 outbound to `authservices.satispay.com`, `staging.authservices.satispay.com` and
`*.amazonaws.com`.
