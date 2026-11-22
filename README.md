# OAuth Client Integration Guide

This guide is for developers building third-party applications on Oceanpay. Your application acts on behalf of an Oceanpay user, with their permission, through OAuth 2.0. It can read their basic profile and their organizations, accounts, orders and payouts, approve or reject payouts that are awaiting approval, and receive webhooks when payouts need a decision or an account's balance runs low.

- [Overview](#overview)
- [Getting credentials](#getting-credentials)
- [Authorizing users](#authorizing-users)
- [Scopes](#scopes)
- [Making requests](#making-requests)
- [Errors](#errors)
- [Endpoints](#endpoints)
- [Webhooks](#webhooks)
- [Verifying webhook signatures](#verifying-webhook-signatures)

## Overview

|                      |                                                                                        |
| -------------------- | -------------------------------------------------------------------------------------- |
| Authorization server | `https://{host}/oauth/...`                                                             |
| API base URL         | `https://{host}/api/client`                                                            |
| Token type           | Bearer access token (`Authorization: Bearer <token>`)                                  |
| Supported grants     | Authorization code (with PKCE for public clients), refresh token, device authorization |
| Webhook signing      | HMAC-SHA256 in the `X-Signature` header                                                |

Every API request is made on behalf of a user. Tokens issued with the client credentials grant carry no user and are rejected by the client API.

## Getting credentials

Oceanpay registers OAuth clients for you. Contact us with:

- your application's name,
- the redirect URI(s) for the authorization code flow, or a note that you need the device flow,
- whether your app is confidential (it has a server that can keep a secret) or public (a mobile or single-page app),
- if you want [webhooks](#webhooks), the HTTPS URL to send them to. Your app has one webhook URL for every organization that authorizes it.

You receive:

| Credential      | Used for                               | Notes                                  |
| --------------- | -------------------------------------- | -------------------------------------- |
| `client_id`     | Every OAuth request                    | A UUID.                                |
| `client_secret` | Exchanging codes and refreshing tokens | Confidential clients only. Shown once. |
| Webhook secret  | Verifying webhook signatures           | Starts with `whsec_`. Shown once.      |

Store both secrets on your server, never in a mobile app or browser code. To change your webhook URL, contact us. If a secret leaks, ask us to rotate it. A rotated webhook secret takes effect immediately: every webhook after the rotation is signed with the new secret only.

## Authorizing users

### Authorization code flow

1. Send the user to the authorization page:

    ```
    GET https://{host}/oauth/authorize
        ?client_id={client_id}
        &redirect_uri={redirect_uri}
        &response_type=code
        &scope=user:read organizations:read payouts:read payouts:approve
        &state={random_state}
    ```

   Public clients must also send `code_challenge` and `code_challenge_method=S256` (PKCE).

   The user signs in to Oceanpay if they need to, sees your application's name and the permissions you ask for, and approves or denies the request.

2. Oceanpay redirects back to `redirect_uri` with `code` and `state`. Check that `state` matches the value you sent.

3. Exchange the code for tokens:

    ```http
    POST https://{host}/oauth/token
    Content-Type: application/x-www-form-urlencoded

    grant_type=authorization_code
    &client_id={client_id}
    &client_secret={client_secret}
    &redirect_uri={redirect_uri}
    &code={code}
    ```

   Public clients send `code_verifier` instead of `client_secret`.

    ```json
    {
        "token_type": "Bearer",
        "expires_in": 31536000,
        "access_token": "eyJ0eXAiOiJKV1Qi...",
        "refresh_token": "def50200..."
    }
    ```

### Refreshing tokens

Access tokens and refresh tokens each last one year. Refresh before the access token expires:

```http
POST https://{host}/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token={refresh_token}
&client_id={client_id}
&client_secret={client_secret}
&scope=user:read organizations:read payouts:read payouts:approve
```

A refresh returns a new access token and a new refresh token, and the old refresh token stops working. Save both new tokens.

### Device authorization flow

For devices without a browser, such as a terminal or a TV app. Your client must be registered as a device client.

1. Request a device code:

    ```http
    POST https://{host}/oauth/device/code
    Content-Type: application/x-www-form-urlencoded

    client_id={client_id}&scope=user:read organizations:read payouts:read
    ```

   The response includes `device_code`, `user_code`, `verification_uri` and `interval`.

2. Show the user `user_code` and ask them to open `verification_uri` (`https://{host}/oauth/device`) on another device, enter the code and approve.

3. Poll the token endpoint every `interval` seconds until the user approves or denies:

    ```http
    POST https://{host}/oauth/token
    Content-Type: application/x-www-form-urlencoded

    grant_type=urn:ietf:params:oauth:grant-type:device_code
    &client_id={client_id}
    &client_secret={client_secret}
    &device_code={device_code}
    ```

   Leave out `client_secret` if your device client is public. While the user hasn't answered, the response is an `authorization_pending` error. If you get `slow_down`, poll less often.

## Scopes

Ask only for the scopes you need. The user sees each one on the authorization page.

| Scope                | Allows                                                                               |
| -------------------- | ------------------------------------------------------------------------------------ |
| `user:read`          | View the user's name and avatar                                                      |
| `organizations:read` | List the user's organizations and view one                                           |
| `accounts:read`      | View an organization's accounts and their balances, and receive low balance webhooks |
| `orders:read`        | View orders                                                                          |
| `orders:write`       | Resend an order's webhook to the merchant                                            |
| `payouts:read`       | View payouts                                                                         |
| `payouts:write`      | Resend a payout's webhook to the merchant                                            |
| `payouts:approve`    | Approve and reject payouts awaiting approval, and receive their webhooks             |

Scopes set the most your app can do. What it can actually do also depends on the user's role in each organization. For example, a token with `payouts:approve` can approve payouts only in organizations where the user is an owner or admin. Each organization from `GET /organizations` includes the user's `role` and `permissions` in it, so your app can tell in advance what will be allowed.

## Making requests

Send the access token as a bearer token and ask for JSON:

```http
GET https://{host}/api/client/organizations
Authorization: Bearer {access_token}
Accept: application/json
```

- **Organizations are identified by their `slug`** in URLs, such as `/api/client/organizations/acme-ltd/accounts`. Get slugs from `GET /organizations`.
- **The user must be a member** of the organization in the URL.
- **The user's email must be verified.** Requests for users with an unverified email return 403.
- **Responses are wrapped in `data`.** Money amounts are decimal strings, such as `"10.25"`. Timestamps are ISO 8601 in UTC.

### Status codes

**Order `status`**

| Value | Meaning |
| ----- | ------- |
| `0`   | Pending |
| `1`   | Paid    |

**Payout `status`**

| Value | Meaning                                                  |
| ----- | -------------------------------------------------------- |
| `0`   | Pending: accepted and waiting to be sent                 |
| `1`   | Successful                                               |
| `2`   | Processing                                               |
| `3`   | Failed                                                   |
| `4`   | Cancelled, which includes rejected and expired approvals |
| `5`   | Reversed                                                 |
| `6`   | Awaiting approval                                        |

**Account `status`:** `pending`, `active` or `inactive`.

## Errors

| Status | When                                                                                                 | Body                                               |
| ------ | ---------------------------------------------------------------------------------------------------- | -------------------------------------------------- |
| 401    | The token is missing, invalid, expired or revoked                                                    | `{"message": "Unauthenticated."}`                  |
| 403    | The token lacks the route's scope                                                                    | `{"message": "Invalid scope(s) provided."}`        |
| 403    | The user isn't a member of the organization, their role can't do this, or their email isn't verified | `{"message": "..."}`                               |
| 404    | No such record in this organization                                                                  | `{"message": "..."}`                               |
| 422    | Validation failed, or the record isn't in a state that allows the action                             | `{"message": "...", "errors": {"field": ["..."]}}` |
| 429    | Too many requests                                                                                    | `{"message": "Too Many Attempts."}`                |

Role checks run before the record is looked up. A user who isn't allowed to approve payouts therefore gets 403 whether or not the payout exists.

## Endpoints

All paths are relative to `https://{host}/api/client`.

### User

#### `GET /user`

Scope: `user:read`. Returns the basic profile of the user who authorized your app. Use `id` to link the Oceanpay user to an account in your app.

```json
{
    "data": {
        "id": 42,
        "name": "Ana Tran",
        "avatar": "https://lh3.googleusercontent.com/a/...",
        "two_factor_enabled": true,
        "created_at": "2026-09-01T08:00:00+00:00"
    }
}
```

- `avatar` is `null` unless the user signed in with Google.
- `two_factor_enabled` tells you whether the user can give an authenticator code. Approving a payout needs one, so ask the user to turn on two-factor authentication in Oceanpay before you offer approvals.

### Organizations

#### `GET /organizations`

Scope: `organizations:read`. Lists every organization the user belongs to.

```json
{
    "data": [
        {
            "id": 12,
            "name": "Acme Ltd",
            "slug": "acme-ltd",
            "is_personal": false,
            "role": "admin",
            "permissions": ["payout:approve", "webhook:resend"],
            "created_at": "2026-09-01T08:00:00+00:00"
        }
    ]
}
```

- `role` is the user's role in the organization: `owner`, `admin`, `developer` or `member`.
- `permissions` lists what the user may do there through this API:

  | Permission       | Lets the user                                         | Roles that have it      |
      | ---------------- | ----------------------------------------------------- | ----------------------- |
  | `payout:approve` | Approve and reject payouts                            | Owner, admin            |
  | `webhook:resend` | Resend an order's or payout's webhook to the merchant | Owner, admin, developer |

- Decide what to offer from `permissions`, not from `role`. Which permissions each role has may change, and new permissions may be added; ignore any you don't know.
- Both describe the user, not your token. Your app still needs the matching scope, such as `payouts:approve`.
- They can change at any time, for example when an owner changes the user's role. Fetch organizations again from time to time, and handle a 403 even when `permissions` allowed the action.

#### `GET /organizations/{organization}`

Scope: `organizations:read`. Returns one organization, in the same shape as an item above.

### Accounts

#### `GET /organizations/{organization}/accounts`

Scope: `accounts:read`. Lists the organization's accounts, newest first. Filter by currency with `?filter[currency_code]=VND`.

```json
{
    "data": [
        {
            "id": 31,
            "currency_code": "VND",
            "balance": "1520.75",
            "frozen_balance": "0.00",
            "low_balance_threshold": "500.00",
            "status": "active",
            "created_at": "2026-09-01T08:00:00+00:00"
        }
    ]
}
```

### Orders

Look up an order by Oceanpay's reference (`trade_no`) or by the merchant's own reference (`out_trade_no`). A reference may contain slashes.

| Method and path                                                                  | Scope          |
| -------------------------------------------------------------------------------- | -------------- |
| `GET /organizations/{organization}/orders/{tradeNo}`                             | `orders:read`  |
| `GET /organizations/{organization}/orders/reference/{reference}`                 | `orders:read`  |
| `POST /organizations/{organization}/orders/{tradeNo}/resend-webhook`             | `orders:write` |
| `POST /organizations/{organization}/orders/reference/{reference}/resend-webhook` | `orders:write` |

```json
{
    "data": {
        "trade_no": "01jd2x...",
        "out_trade_no": "INV-2026-0042",
        "amount": "50000.00",
        "paid_amount": "50000.00",
        "currency_code": "VND",
        "subject": "Order #42",
        "status": 1,
        "paid_at": "2026-11-13T09:30:00+00:00",
        "created_at": "2026-11-13T09:00:00+00:00"
    }
}
```

**Resending a webhook** sends the merchant's own `order.paid` webhook to the order's `notify_url` again. It does not send anything to your application. Only paid orders with a `notify_url` can be resent, otherwise the response is 422. The user needs a role that may resend webhooks. A successful resend returns `202 Accepted`:

```json
{
    "data": {
        "id": "9d6c1e9a-...",
        "event_type": "order.paid",
        "url": "https://merchant.example/webhooks",
        "status": "pending",
        "created_at": "2026-11-22T10:00:00+00:00"
    }
}
```

### Payouts

Look up a payout by Oceanpay's reference (`transfer_no`) or by the merchant's reference (`out_transfer_no`).

| Method and path                                                                   | Scope             |
| --------------------------------------------------------------------------------- | ----------------- |
| `GET /organizations/{organization}/payouts/{transferNo}`                          | `payouts:read`    |
| `GET /organizations/{organization}/payouts/reference/{reference}`                 | `payouts:read`    |
| `POST /organizations/{organization}/payouts/{transferNo}/resend-webhook`          | `payouts:write`   |
| `POST /organizations/{organization}/payouts/reference/{reference}/resend-webhook` | `payouts:write`   |
| `POST /organizations/{organization}/payouts/{transferNo}/approve`                 | `payouts:approve` |
| `POST /organizations/{organization}/payouts/reference/{reference}/approve`        | `payouts:approve` |
| `POST /organizations/{organization}/payouts/{transferNo}/reject`                  | `payouts:approve` |
| `POST /organizations/{organization}/payouts/reference/{reference}/reject`         | `payouts:approve` |

```json
{
    "data": {
        "transfer_no": "01jd2y...",
        "out_transfer_no": "PAYOUT/2026/001",
        "amount": "10.25",
        "total_fees": "0.50",
        "total_amount": "10.75",
        "currency_code": "VND",
        "payee_name": "Recipient",
        "payee_account_last4": "6789",
        "status": 6,
        "approval_expires_at": "2026-11-23T10:00:00+00:00",
        "approval_decided_at": null,
        "paid_at": null,
        "created_at": "2026-11-22T10:00:00+00:00"
    }
}
```

Only the last four digits of the payee's account are shown.

**Resending a payout webhook** works like resending an order's. It sends the merchant's `payout.succeeded` or `payout.failed` webhook to the payout's `notify_url`, and only for finished payouts with a `notify_url`.

#### Approving a payout

A payout at or above its account's approval threshold waits for an owner or admin of the organization (`status` `6`). If nobody approves it within 24 hours (`approval_expires_at`), it is cancelled and refunded.

To approve, the owner or admin must give your app the current 6-digit code from their authenticator app, so they need two-factor authentication turned on. Approving without that code isn't possible, so prompt them for it every time.

```http
POST /api/client/organizations/acme-ltd/payouts/01jd2y.../approve
Authorization: Bearer {access_token}
Content-Type: application/json

{ "totp": "123456" }
```

- **Success:** the updated payout, with `status` `0` (pending). It is then sent like any other payout.
- **403:** the user isn't an owner or admin.
- **422:** the code is wrong, the payout is no longer awaiting approval, or the approval window has passed.
- **Rate limit:** 10 approve requests per minute.

#### Rejecting a payout

```http
POST /api/client/organizations/acme-ltd/payouts/01jd2y.../reject
Authorization: Bearer {access_token}
```

No body is needed. The payout is cancelled (`status` `4`) and its amount and fees are returned to the account. Only owners and admins can reject.

## Webhooks

Webhooks tell your app about events in the organizations that authorized it. They're all sent to your client's one webhook URL, which we set when we register your client. There's nothing to subscribe to through the API.

### Who receives webhooks

Your app receives an organization's event when at least one of its members authorized your app with the event's scope and has the role the event needs:

| Event                       | Token scope       | User's role in the organization |
| --------------------------- | ----------------- | ------------------------------- |
| `payout.approval_requested` | `payouts:approve` | Owner or admin                  |
| `payout.approval_decided`   | `payouts:approve` | Owner or admin                  |
| `account.low_balance`       | `accounts:read`   | Owner or admin                  |

- Your app gets each event once, however many of the organization's members authorized it.
- To pass an event on to your users, send it only to those who qualify: `payout:approve` in `permissions` for the payout events, and a `role` of `owner` or `admin` for `account.low_balance`.
- Your app gets every event type it qualifies for. Answer 2xx and ignore any type you don't use.
- The user's authorization counts while its access token is valid, or while its refresh token can still renew it.

### Event types

| Type                        | Sent when                                                                                                                                                                                                                                                                                               |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `payout.approval_requested` | A payout is held because it is at or above its account's approval threshold.                                                                                                                                                                                                                            |
| `payout.approval_decided`   | A held payout stops waiting. Either an owner or admin approved or rejected it, in your app, in the Oceanpay dashboard or in another app, or it expired after 24 hours.                                                                                                                                  |
| `account.low_balance`       | An account's balance falls below its low balance threshold. It's sent once per drop. It's sent again only after the balance is back at or above the threshold and falls below it again. A changed threshold doesn't alert at once; the next debit that leaves the balance below the new threshold does. |

`payout.approval_requested` is skipped if the payout was already decided before the webhook was queued. `payout.approval_decided` is still sent.

### Request

Oceanpay sends a `POST` with a JSON body and these headers:

```http
POST /webhooks/oceanpay HTTP/1.1
Content-Type: application/json
X-Signature: t=1795304519,v1=5257a869e7ecebeda32affa62cdca3fa51cad7e77a0e56ff536d0ce8e108d8bd
```

### Payload

```json
{
    "id": "evt_01jd3b2c4d5e6f7g8h9j0k1m2n",
    "type": "payout.approval_requested",
    "created_at": "2026-11-22T10:00:00+00:00",
    "organization": {
        "id": 12,
        "slug": "acme-ltd"
    },
    "data": {
        "object": {
            "transfer_no": "01jd2y...",
            "out_transfer_no": "PAYOUT/2026/001",
            "amount": "10.25",
            "total_fees": "0.50",
            "total_amount": "10.75",
            "currency_code": "VND",
            "payee_name": "Recipient",
            "payee_account_last4": "6789",
            "status": 6,
            "approval_expires_at": "2026-11-23T10:00:00+00:00",
            "approval_decided_at": null,
            "paid_at": null,
            "created_at": "2026-11-22T10:00:00+00:00"
        }
    }
}
```

`data.object` has the same fields as the payout from `GET /payouts/{transferNo}`, as it was when the event happened. `payout.approval_decided` also has `data.outcome`:

| `outcome`  | Meaning                                                       | Payout `status` |
| ---------- | ------------------------------------------------------------- | --------------- |
| `approved` | An owner or admin approved it. It is being sent.              | `0`             |
| `rejected` | An owner or admin rejected it. It was cancelled and refunded. | `4`             |
| `expired`  | Nobody approved it in time. It was cancelled and refunded.    | `4`             |

```json
{
    "id": "evt_01jd3c...",
    "type": "payout.approval_decided",
    "created_at": "2026-11-22T10:05:00+00:00",
    "organization": { "id": 12, "slug": "acme-ltd" },
    "data": {
        "outcome": "approved",
        "object": {
            "transfer_no": "01jd2y...",
            "status": 0,
            "approval_decided_at": "2026-11-22T10:05:00+00:00",
            "...": "..."
        }
    }
}
```

#### `account.low_balance`

`data.balance` and `data.threshold` are the values at the moment the balance fell below the threshold. `data.object` is the account in the same shape as `GET /accounts`, taken when the webhook was queued. More debits may already have landed by then, so its `balance` can be lower than `data.balance`.

```json
{
    "id": "evt_01jd3d...",
    "type": "account.low_balance",
    "created_at": "2026-11-22T11:00:00+00:00",
    "organization": { "id": 12, "slug": "acme-ltd" },
    "data": {
        "balance": "400.00",
        "threshold": "500.00",
        "object": {
            "id": 31,
            "currency_code": "VND",
            "balance": "400.00",
            "frozen_balance": "0.00",
            "low_balance_threshold": "500.00",
            "status": "active",
            "created_at": "2026-09-01T08:00:00+00:00"
        }
    }
}
```

### Responding and retries

- Answer with any 2xx status within 10 seconds. Do the slow work after you respond.
- Any other status, a redirect, a timeout or a connection error counts as a failure. Redirects aren't followed.
- A failed webhook is retried after 1 minute, 5 minutes, 30 minutes, 2 hours, 6 hours, 12 hours and 24 hours. That's 8 attempts over about 3.5 days. Each retry sends the same payload with the same event `id`.
- Use the event `id` to ignore webhooks you have already handled. Retries of an event keep its `id`.
- Events can arrive out of order. When the order matters, compare the payout's `status`, or fetch the payout or account from the API.

### When webhooks stop

Before queuing and again before each attempt, Oceanpay checks that your app can still receive the event. Webhooks for an organization stop, and pending ones fail without being sent, when no member who authorized your app still qualifies. That happens when each of them:

- no longer has the role the event needs, or has left the organization,
- revoked your app's authorization, or let both its access token and its refresh token expire,
- holds a token without the event's scope.

Webhooks also stop for every organization when your OAuth client is revoked, its webhook URL is removed, it has no webhook secret, or the URL resolves to a private or reserved address.

## Verifying webhook signatures

Verify every webhook before you trust it. The `X-Signature` header has the form `t={timestamp},v1={signature}`:

- `t` is the Unix time the webhook was signed. It changes on every attempt.
- `v1` is the lowercase hex HMAC-SHA256 of `{t}.{raw request body}`, keyed with your webhook secret.

To verify:

1. Read the **raw** request body exactly as received. Don't parse and re-encode the JSON first, or the bytes will differ.
2. Split the header on `,`, then each part on its first `=`, ignoring spaces. Take `t` and every `v1` value.
3. Compute `HMAC-SHA256(secret, t + "." + rawBody)` as hex.
4. Compare it with each `v1` value using a constant-time comparison. Accept the webhook if any one matches.
5. Reject it if `t` is more than 5 minutes from your current time, to block replays.

### Python (FastAPI)

```python
import hashlib
import hmac
import json
import os
import time

from fastapi import FastAPI, HTTPException, Request

app = FastAPI()


def verify_oceanpay_webhook(raw_body: bytes, header: str, secret: str, tolerance: int = 300) -> bool:
    timestamp = None
    signatures = []

    for part in header.split(","):
        key, _, value = part.strip().partition("=")

        if key == "t" and value.isdigit():
            timestamp = int(value)
        elif key == "v1" and value:
            signatures.append(value)

    if timestamp is None or abs(time.time() - timestamp) > tolerance:
        return False

    expected = hmac.new(
        secret.encode(),
        f"{timestamp}.".encode() + raw_body,
        hashlib.sha256,
    ).hexdigest()

    return any(hmac.compare_digest(expected, signature) for signature in signatures)


@app.post("/webhooks/oceanpay")
async def oceanpay_webhook(request: Request):
    # Read the raw bytes: verification needs exactly what was signed, not re-encoded JSON.
    raw_body = await request.body()

    if not verify_oceanpay_webhook(raw_body, request.headers.get("X-Signature", ""), os.environ["OCEANPAY_WEBHOOK_SECRET"]):
        raise HTTPException(status_code=400, detail="Invalid signature")

    event = json.loads(raw_body)
    # Queue the event for processing and respond quickly.

    return {"received": True}
```

### Node.js (Express)

```js
import crypto from 'node:crypto';
import express from 'express';

function verifyOceanpayWebhook(rawBody, header, secret, tolerance = 300) {
    let timestamp = null;
    const signatures = [];

    for (const part of (header ?? '').split(',')) {
        const [key, value = ''] = part.trim().split(/=(.*)/s);
        if (key === 't' && /^\d+$/.test(value)) timestamp = Number(value);
        else if (key === 'v1' && value) signatures.push(value);
    }

    if (
        timestamp === null ||
        Math.abs(Date.now() / 1000 - timestamp) > tolerance
    )
        return false;

    const expected = Buffer.from(
        crypto
            .createHmac('sha256', secret)
            .update(`${timestamp}.${rawBody}`)
            .digest('hex'),
    );

    return signatures.some((signature) => {
        const candidate = Buffer.from(signature);
        return (
            candidate.length === expected.length &&
            crypto.timingSafeEqual(candidate, expected)
        );
    });
}

const app = express();

// Keep the raw body: verification needs the exact bytes that were signed.
app.post(
    '/webhooks/oceanpay',
    express.raw({ type: 'application/json' }),
    (req, res) => {
        if (
            !verifyOceanpayWebhook(
                req.body.toString('utf8'),
                req.get('X-Signature'),
                process.env.OCEANPAY_WEBHOOK_SECRET,
            )
        ) {
            return res.sendStatus(400);
        }

        const event = JSON.parse(req.body);
        res.sendStatus(200);
        // Handle the event after responding.
    },
);
```

### Test vector

Use this to check your implementation:

|           |                                                                                    |
| --------- | ---------------------------------------------------------------------------------- |
| Secret    | `whsec_test`                                                                       |
| Timestamp | `1795304519`                                                                       |
| Raw body  | `{"id":"evt_1"}`                                                                   |
| Header    | `t=1795304519,v1=aa63c86f4e0415ba1c39e0eb926d2246042a4dc8768fd36568dc9bb79da52d18` |
