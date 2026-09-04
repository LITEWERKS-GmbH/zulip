# Implement self-hosted iOS Declarative Web Push (iOS/iPadOS 18.4+)

## Status and baseline

- **Repository:** `LITEWERKS-GmbH/zulip`
- **Implementation branch:** `litewerks-webpush`
- **Upstream baseline:** Zulip Server `12.2`
- **Baseline commit:** `1e73e1d754761b73c18135a3f25d0673f31cd8b3`
- **Research/review date:** 2026-09-04
- **Production status:** Not implemented and not deployable yet.
- **Target client for v1:** An authenticated Zulip web app added to the iPhone/iPad Home Screen, running iOS/iPadOS 18.4 or newer.

This issue is the implementation handoff and acceptance contract for the downstream Litewerks Web Push feature. It incorporates a second, independent critical review of the initial design. The plan deliberately favors an isolated, upgradeable patch over broad changes to Zulip's notification policy.

---

## Objective

Provide reliable push notifications from the self-hosted Litewerks Zulip server to iPhones and iPads without:

- Zulip's hosted Mobile Push Notification Service;
- a fork of the native Zulip Flutter application;
- Apple Developer Program membership;
- APNs certificates or keys;
- Firebase;
- TestFlight or App Store distribution; or
- a service worker in the initial iOS 18.4+ implementation.

The resulting user journey should be:

1. Open the self-hosted Zulip URL in Safari.
2. Use **Share → Add to Home Screen**.
3. Open the installed Home Screen web app.
4. Open Zulip notification settings and select **Enable notifications on this device**.
5. Approve the iOS notification permission prompt.
6. Receive lock-screen notifications while the web app is backgrounded or closed.
7. Tap a notification and open the correct Zulip conversation/message.

The feature must survive normal Zulip deployments because it is maintained in the Git branch and deployed with Zulip's supported `upgrade-zulip-from-git` workflow. Runtime secrets stay under `/etc/zulip`, and subscriptions stay in PostgreSQL.

---

## Verified platform facts

The implementation is based on the following current platform behavior:

1. WebKit introduced **Declarative Web Push** on iOS and iPadOS 18.4. It allows a user-visible push notification to be described by standardized JSON and displayed without executing a service worker.
2. WebKit exposes `window.pushManager` for managing a push subscription without a service worker.
3. Apple uses APNs internally for Web Push, but the website operator does not need Apple Developer Program membership or APNs credentials.
4. iPhone/iPad Web Push is available to a web app added to the Home Screen. Permission must be requested following a direct user action.
5. A declarative payload requires:
   - top-level `"web_push": 8030`;
   - a non-empty notification `title`; and
   - a `navigate` URL.
6. Web Push payloads are encrypted using the standard Web Push message-encryption protocol (`aes128gcm`). Apple receives routing metadata and encrypted bytes, not plaintext notification content.
7. The HTTP Web Push request must include a `TTL` header. A `Topic` header may replace an outstanding message for the same subscription and topic.
8. Without a service worker, there is currently no `pushsubscriptionchange` callback that can notify our server immediately when WebKit rotates a subscription. The client must reconcile on app launch, and the server must remove endpoints that return a permanent expiration response.

Primary references are listed at the end of this issue.

---

## Scope

### Included in v1

- iOS/iPadOS 18.4+ Declarative Web Push.
- Home Screen web-app installability.
- Explicit, user-gesture-driven notification permission and subscription.
- Device-specific enable/disable/status UI inside Zulip's existing notification settings.
- Standard VAPID authentication and RFC 8291 payload encryption.
- Multiple Web Push subscriptions per Zulip user.
- Safe transfer of a browser subscription when the same Home Screen install changes accounts.
- Reuse of Zulip's existing:
  - DM/mention/channel/followed-topic notification preferences;
  - online/offline and idle behavior;
  - muted sender/channel/topic handling;
  - do-not-disturb logic;
  - read/deleted-message rechecks;
  - notification queue and worker;
  - message rendering and deep-link semantics.
- Deep links into the relevant message/conversation.
- Permanent endpoint cleanup and bounded transient-error handling.
- VAPID configuration, diagnostics, tests, deployment documentation, and manual iPhone acceptance testing.
- Compatibility with organizations that enable Zulip's “Require end-to-end encryption for push notifications” setting, after documenting and testing the separate standards-based Web Push encryption model.

### Explicitly excluded from v1

- Android, desktop Chrome/Edge/Firefox, or iOS 16.4–18.3 support.
- Traditional service-worker-based Push API delivery.
- Offline caching or an offline-capable Zulip client.
- Background data synchronization.
- Notification actions, inline replies, or silent push.
- A native app fork.
- Per-organization dynamic PWA branding.
- Automatic production deployment from upstream.
- Replacing Zulip's existing hosted/native push implementation.
- Broad refactoring of Zulip notification policy unrelated to Web Push.

---

## Architectural principles

1. **Web Push is a transport, not a new notification policy.**  
   Zulip remains responsible for deciding whether a message deserves a push. The new code only registers Web Push devices and delivers the notification after Zulip's existing eligibility checks.

2. **Keep native push behavior unchanged.**  
   APNs/FCM/bouncer logic must continue to operate exactly as before when configured.

3. **Web Push-only operation must be valid.**  
   The server must consider push configured when valid Web Push/VAPID configuration exists even if Zulip's hosted service, APNs, and FCM are all disabled.

4. **Patch the stable branch, not the live deployment.**  
   All source changes live on `litewerks-webpush`. Do not edit `/home/zulip/deployments/current` manually.

5. **Minimize long-term merge surface.**  
   Add isolated modules and small integration calls. Do not copy or rewrite the entire notification subsystem.

6. **Fail closed.**  
   Missing or inconsistent VAPID configuration disables Web Push cleanly. It must not crash the mobile notification worker or expose a partial UI that cannot work.

7. **Treat subscription endpoints as secrets and as untrusted outbound destinations.**  
   They are bearer capability URLs and also an SSRF input surface. Never log them. Validate them strictly and restrict v1 to Apple's push domains.

8. **No secrets in Git.**  
   The VAPID private key belongs in a root-controlled file under `/etc/zulip`, referenced by configuration.

---

## Target architecture

```text
iPhone/iPad Home Screen web app
          │
          │ explicit user action
          ▼
window.pushManager.subscribe()
          │
          │ endpoint + p256dh + auth
          ▼
Authenticated Zulip registration API
          │
          ▼
WebPushSubscription row in PostgreSQL
          │
          │ counts as a registered push device
          ▼
Existing Zulip notification eligibility logic
          │
          ▼
missedmessage_mobile_notifications queue
          │
          ▼
Existing read/deleted/settings recheck
          │
          ▼
Transport-neutral notification data
          │
          ▼
Litewerks Web Push adapter
          │
          │ VAPID + aes128gcm, no redirects
          ▼
Apple Web Push endpoint (*.push.apple.com)
          │
          ▼
Declarative notification displayed by WebKit
          │
          ▼
Required same-origin navigate URL opens Zulip
```

---

# Implementation plan

## Phase 0 — Freeze the baseline and run an Apple transport spike

Before changing Zulip's notification pipeline, prove the smallest complete transport against a real iPhone.

### Tasks

- [ ] Confirm the server upgrade completed successfully on Zulip `12.2`.
- [ ] Confirm `litewerks-webpush` is still rebased directly on tag `12.2`.
- [ ] Create a development environment from `LITEWERKS-GmbH/zulip:litewerks-webpush`.
- [ ] Generate a non-production VAPID P-256 key pair.
- [ ] Add only the minimum temporary page/code required to:
  - obtain a subscription using `window.pushManager.subscribe(...)`;
  - print/copy the subscription in a controlled development environment;
  - send a minimal declarative payload to it using a candidate Python sender.
- [ ] Test on a physical iPhone running iOS 18.4+ with the site added to the Home Screen.
- [ ] Verify:
  - permission can only be requested following the button action;
  - the app can subscribe without a service worker;
  - Apple accepts the VAPID JWT;
  - the push arrives while the PWA is closed;
  - tapping it navigates to the supplied URL;
  - `aes128gcm`, `TTL`, and the proposed VAPID `sub` value work.
- [ ] Record the exact successful request behavior in the PR/issue.
- [ ] Remove temporary diagnostic UI before merging.

### Why this is mandatory

`pywebpush` is the leading Python implementation and currently has release `2.3.0`, but Apple-specific VAPID failures have been reported by users. We must prove the exact dependency/configuration against Apple's current service before coupling it to Zulip. If the spike fails, stop and fix/replace only the isolated sender adapter rather than building the rest on an unverified transport.

### Exit criterion

A minimal encrypted Declarative Web Push reaches a real iPhone from a development Zulip origin without a service worker.

---

## Phase 1 — Add the pinned server dependency and VAPID configuration

### Dependency

Use a narrow adapter around a pinned `pywebpush` version rather than implementing RFC 8291/VAPID cryptography ourselves.

- [ ] Add `pywebpush==2.3.*` (or the exact version validated by Phase 0) to the production dependency group in `pyproject.toml`.
- [ ] Update `uv.lock`.
- [ ] Add any required mypy stubs or narrowly scoped `ignore_missing_imports` configuration.
- [ ] Increment `PROVISION_VERSION` if Zulip's provisioning rules require it.
- [ ] Run provisioning from a clean environment to prove production installation includes the dependency.

### Settings

Add configuration with names consistent with Zulip conventions:

```python
WEB_PUSH_VAPID_PRIVATE_KEY_FILE: str | None = None
WEB_PUSH_VAPID_SUBJECT: str | None = None
WEB_PUSH_ALLOWED_HOST_SUFFIXES = ("push.apple.com",)
WEB_PUSH_MAX_SUBSCRIPTIONS_PER_USER = 10
WEB_PUSH_TTL_SECONDS = 24 * 60 * 60
```

The exact names may change during implementation, but the semantics must remain.

- [ ] Read the VAPID private key from a file, not an inline committed string.
- [ ] Derive the public application-server key from the private key at startup.
- [ ] Expose only the public key to the authenticated web client.
- [ ] Validate `WEB_PUSH_VAPID_SUBJECT` as either `mailto:...` or an HTTPS URL.
- [ ] Add `web_push_configured()` that returns true only when the complete, valid configuration is available.
- [ ] Ensure invalid configuration causes a clear startup/worker warning without logging key material.
- [ ] Document a production location such as:

```text
/etc/zulip/web-push-vapid-private.pem
```

with restrictive ownership/permissions.

### Key rotation

- [ ] Calculate a non-secret fingerprint of the VAPID public key.
- [ ] Return that fingerprint/public key in the client configuration.
- [ ] On the client, detect when an existing subscription was created with a different application-server key.
- [ ] Unsubscribe and create/register a replacement.
- [ ] Do not implement routine automatic key rotation in v1; rotation is an explicit runbook operation requiring clients to revisit/open the PWA.

---

## Phase 2 — Add the persistent subscription model and migration

Add a dedicated model to `zerver/models/push_notifications.py` and export it through `zerver/models/__init__.py`.

### Proposed minimal model

```python
class WebPushSubscription(models.Model):
    user = models.ForeignKey(UserProfile, on_delete=CASCADE, db_index=True)
    endpoint = models.TextField()
    endpoint_sha256 = models.CharField(max_length=64, unique=True)
    p256dh = models.CharField(max_length=256)
    auth = models.CharField(max_length=128)
    expiration_time = models.DateTimeField(null=True)
    vapid_key_fingerprint = models.CharField(max_length=64)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    last_seen_at = models.DateTimeField()
```

Exact field lengths should be confirmed against actual Apple subscriptions and tests.

### Requirements

- [ ] `endpoint_sha256` is globally unique so one browser subscription cannot remain assigned to two Zulip users.
- [ ] The raw endpoint is stored because delivery requires it, but it is never used in logs, analytics, error messages, or URLs generated by Zulip.
- [ ] Validate and decode `p256dh` and `auth` as Base64URL at registration time.
- [ ] Validate expected decoded key lengths/format before persistence.
- [ ] Parse `expirationTime` from the browser safely; allow `null`.
- [ ] Cap active subscriptions per user (default 10). On overflow, reject or deterministically evict the oldest inactive subscription; choose one behavior and test it.
- [ ] Add efficient indexes for:
  - subscriptions by user;
  - globally unique endpoint hash; and
  - optional stale-subscription cleanup by `last_seen_at`.
- [ ] Use a normal forward migration with no production data migration.
- [ ] Confirm migration rollback is safe before any production subscriptions exist.
- [ ] Never store a user-agent string unless an operational requirement is demonstrated.

### Account transfer semantics

A Web Push subscription belongs to the installed origin/browser instance, not permanently to a Zulip account.

- [ ] Registration must atomically upsert by `endpoint_sha256`.
- [ ] If the endpoint already belongs to the same user, update keys, expiry, fingerprint, and `last_seen_at`.
- [ ] If it belongs to another user, atomically transfer it to the currently authenticated user.
- [ ] Do not reveal whether it previously belonged to another user.
- [ ] Add race tests for two accounts trying to claim the same endpoint.
- [ ] Ensure a stale DELETE from the old account cannot remove a row after ownership has transferred, by scoping deletion to both authenticated user and endpoint hash.

---

## Phase 3 — Add authenticated APIs

Follow Zulip's typed endpoint and authentication patterns.

### Proposed endpoints

#### Get Web Push capability/configuration

```text
GET /api/v1/users/me/web_push
```

Response:

```json
{
  "configured": true,
  "vapid_public_key": "...",
  "vapid_key_fingerprint": "...",
  "supported_client": "ios-declarative-18.4+"
}
```

Do not return stored endpoint/key material.

#### Register or reconcile a subscription

```text
POST /api/v1/users/me/web_push_subscription
```

Body:

```json
{
  "endpoint": "https://...",
  "keys": {
    "p256dh": "...",
    "auth": "..."
  },
  "expiration_time": null,
  "vapid_key_fingerprint": "..."
}
```

#### Disable/delete the current subscription

```text
DELETE /api/v1/users/me/web_push_subscription
```

Pass the endpoint or its client-computed SHA-256 in the JSON body, not in a query string.

#### Send a test notification

```text
POST /api/v1/users/me/web_push/test
```

This must target only the authenticated user's specified/current subscription.

### API safeguards

- [ ] Human users only; bots cannot register subscriptions.
- [ ] Standard Zulip session/API authentication and CSRF protections.
- [ ] Idempotent POST behavior.
- [ ] Strict JSON schema and size limits.
- [ ] Conservative rate limits:
  - registration/reconciliation: sufficient for normal launch reconciliation but resistant to abuse;
  - test push: low per-user limit.
- [ ] Generic error responses that do not expose endpoint ownership or key details.
- [ ] No endpoint/key material in access logs. Verify reverse-proxy and application logs.
- [ ] Tests for malformed Base64URL, oversized values, missing fields, expired values, duplicate endpoints, account transfer, and unauthenticated access.

---

## Phase 4 — Make Zulip installable as a Home Screen web app

### Manifest

Add `static/app.webmanifest` (or the location that best fits Zulip's static pipeline) and link it from the primary app template.

Initial manifest semantics:

```json
{
  "id": "/",
  "name": "Zulip",
  "short_name": "Zulip",
  "start_url": "/",
  "scope": "/",
  "display": "standalone",
  "background_color": "...",
  "theme_color": "...",
  "icons": [
    {"src": "...", "sizes": "192x192", "type": "image/png"},
    {"src": "...", "sizes": "512x512", "type": "image/png"}
  ]
}
```

### Requirements

- [ ] Use existing Zulip assets where valid; add only missing required icon sizes.
- [ ] Serve the manifest with the correct content type.
- [ ] Keep the existing `mobile-web-app-capable` meta tag in v1. A previous upstream manifest PR was closed after maintainers requested more real-device validation and questioned removing legacy metadata.
- [ ] Confirm login redirects and root routing remain within `scope`.
- [ ] Confirm the installed PWA uses standalone presentation.
- [ ] Do not add service-worker registration or offline caches.
- [ ] Do not cache authenticated Zulip content outside Zulip's current behavior.
- [ ] Add automated template/static checks and real-device install validation.
- [ ] Use generic Zulip naming for v1; dynamic realm branding can be a later isolated enhancement.

---

## Phase 5 — Implement the frontend subscription state machine

Create an isolated module, for example `web/src/web_push.ts`, and integrate a small status/control panel under **Settings → Notifications → Mobile message notifications**.

### Capability detection

Use feature detection, not user-agent parsing:

```typescript
const isStandalone =
    window.matchMedia("(display-mode: standalone)").matches ||
    ("standalone" in navigator && navigator.standalone === true);

const supported =
    window.isSecureContext &&
    isStandalone &&
    "Notification" in window &&
    "pushManager" in window;
```

Type the WebKit-specific API locally without weakening global TypeScript checks.

### UI states

- Not configured on the server.
- Not installed as a Home Screen app.
- Unsupported browser/platform.
- Permission not requested.
- Enabling.
- Enabled and reconciled.
- Permission denied.
- Existing subscription stale or using old VAPID key.
- Temporary registration/delivery error.
- Disabled.

### Enable flow

- [ ] Trigger only from a visible user action.
- [ ] Fetch current public VAPID configuration.
- [ ] Request `Notification` permission.
- [ ] Convert the VAPID public key to the expected `Uint8Array`.
- [ ] Call:

```typescript
await window.pushManager.subscribe({
    userVisibleOnly: true,
    applicationServerKey,
});
```

- [ ] Validate `subscription.toJSON()` contains endpoint, `p256dh`, and `auth`.
- [ ] POST the subscription to the authenticated endpoint.
- [ ] Mark the local UI enabled only after server registration succeeds.
- [ ] Offer **Send test notification** immediately.
- [ ] Present actionable errors without exposing secrets.

### Reconciliation flow

Because no service worker receives `pushsubscriptionchange` in v1:

- [ ] On PWA startup after authentication, call `window.pushManager.getSubscription()`.
- [ ] Reconcile again whenever notification settings are opened.
- [ ] If a local subscription exists, idempotently POST it to refresh ownership, keys, expiry, VAPID fingerprint, and `last_seen_at`.
- [ ] If the VAPID key/fingerprint changed, unsubscribe and resubscribe.
- [ ] If permission is denied, do not repeatedly prompt.
- [ ] Keep launch reconciliation bounded and non-blocking; Zulip must remain usable if it fails.

### Disable/logout flow

Before destroying the authenticated session:

1. Capture the current subscription JSON.
2. Best-effort DELETE the row scoped to the current authenticated user.
3. Call `subscription.unsubscribe()` regardless of DELETE success.
4. Complete logout even if cleanup fails.
5. A server row left after local unsubscribe must be deleted when Apple later returns a permanent endpoint-expired response.

Add tests for server-delete failure, browser-unsubscribe failure, logout, and account switching.

### Existing settings

- [ ] Continue using Zulip's existing mobile notification trigger matrix and:
  - `enable_offline_push_notifications`;
  - `enable_online_push_notifications`.
- [ ] Do not introduce a second Web Push-specific DM/mention/channel policy matrix.
- [ ] The new panel controls whether this physical web-app installation is registered, not which message types are eligible.

---

## Phase 6 — Make Web Push subscriptions count as registered push devices

This is a mandatory integration point that was missing from the initial high-level proposal.

Zulip currently suppresses push eligibility before queueing when a user has neither:

- a modern `Device` with a registered push token; nor
- a legacy `PushDeviceToken`.

A Web Push-only user would otherwise never reach the notification worker.

### Tasks

- [ ] Introduce a small shared query helper representing **has any registered push transport**:
  - modern E2EE mobile device;
  - legacy APNs/FCM token;
  - Web Push subscription.
- [ ] Replace duplicated existence expressions with this helper where practical.
- [ ] Add `WebPushSubscription` to the `has_push_device_registered` annotation in `zerver/actions/message_send.py`.
- [ ] Audit all uses of:
  - `has_push_device_registered`;
  - `push_device_registered_user_ids`;
  - `PushDeviceToken.objects.filter(...)`;
  - `Device.objects.filter(...push_token_id__isnull=False...)`;
  - `active_mobile_push_notification`.
- [ ] Update message-edit paths so newly notifiable edits work for Web Push-only users.
- [ ] Update mark-read/delete/notification-clear paths so internal active-push state is cleared correctly for Web Push-only users.
- [ ] Add tests proving a user with only a `WebPushSubscription` receives the same enqueue decision as a user with a native push device.
- [ ] Add negative tests proving users with no push transport are still excluded.

### Audit command

Use an exhaustive repository search similar to:

```bash
rg -n \
  'PushDeviceToken|push_token_id__isnull|has_push_device_registered|push_device_registered_user_ids|active_mobile_push_notification' \
  zerver
```

Every hit must be classified in the PR description as:

- changed;
- intentionally unchanged; or
- test-only/documentation.

---

## Phase 7 — Integrate Web Push into configured-state and realm lifecycle

Zulip's `realm.push_notifications_enabled` controls whether mobile notification settings are available. It is maintained by the server and initialized/updated based on whether push infrastructure is configured.

### Tasks

- [ ] Extend `push_notifications_configured()` to include valid Web Push configuration.
- [ ] Do **not** redefine `sends_notifications_directly()` if other code assumes it specifically means APNs+FCM; instead introduce a clearly named helper or call `push_notifications_configured()` where general semantics are intended.
- [ ] Update `initialize_push_notifications()` so Web Push-only configuration:
  - enables `push_notifications_enabled` for existing realms;
  - leaves the end timestamp unset;
  - disables it again only if no supported push transport remains configured.
- [ ] Audit realm creation and import paths currently using `sends_notifications_directly()` and initialize new/imported realms correctly for Web Push-only operation.
- [ ] Ensure state data presented to the web client enables the existing Mobile column/settings.
- [ ] Ensure disabling Zulip's hosted mobile service does not disable Web Push.
- [ ] Add startup/configuration tests for:
  - no push configuration;
  - Web Push only;
  - Zulip bouncer only;
  - direct APNs/FCM only;
  - Web Push plus another transport;
  - malformed Web Push configuration.

---

## Phase 8 — Build an isolated, hardened Web Push sender

Create `zerver/lib/web_push.py` or an equivalently isolated module.

### Endpoint validation and SSRF protection

The subscription endpoint is user-provided input that the backend will POST to. v1 must restrict it to Apple.

- [ ] Require HTTPS.
- [ ] Reject usernames/passwords in the URL.
- [ ] Reject fragments.
- [ ] Reject IP literals.
- [ ] Reject non-default ports.
- [ ] Normalize the hostname safely.
- [ ] Allow only:
  - `push.apple.com`; or
  - a hostname ending in `.push.apple.com`.
- [ ] Disable HTTP redirects.
- [ ] Use explicit short connect/read timeouts.
- [ ] Never use a caller-provided proxy override.
- [ ] Document the production firewall requirement to allow outbound HTTPS to `*.push.apple.com`.

Do not broaden the endpoint allowlist for Chrome/Firefox until those transports are intentionally supported and tested.

### HTTP client behavior

`pywebpush` uses `requests` synchronously by default. Wrap it so our behavior is explicit:

- [ ] Provide a controlled `requests.Session` or adapter that forces `allow_redirects=False`.
- [ ] Use an explicit timeout (for example, approximately 5 seconds connect and 10 seconds read; finalize through testing).
- [ ] Do not rely on pywebpush's large default timeout.
- [ ] Keep retries in Zulip's queue/worker layer rather than hidden inside requests.
- [ ] Redact all sensitive fields from exceptions before logging.
- [ ] Pin the dependency version proven in Phase 0.

### VAPID

- [ ] Use ES256.
- [ ] Set the audience to the exact origin of the push endpoint.
- [ ] Use a valid `mailto:` or HTTPS subject.
- [ ] Keep token expiration below 24 hours; pywebpush's 12-hour default is acceptable.
- [ ] Treat 401/403 as configuration/signing failures, not as proof that all user subscriptions are invalid.

---

## Phase 9 — Build the declarative notification payload

### Required shape

```json
{
  "web_push": 8030,
  "notification": {
    "title": "…",
    "body": "…",
    "navigate": "https://chat.example.com/#narrow/…/near/123",
    "tag": "…",
    "icon": "https://chat.example.com/static/…"
  }
}
```

### Message formatting

- [ ] Extract or refactor a transport-neutral title/body builder from Zulip's existing push notification formatting. Do not create a second, subtly different Markdown/recipient rendering implementation.
- [ ] Preserve localization using the recipient user's language.
- [ ] Respect inaccessible/deactivated sender handling already present in native push.
- [ ] Convert rendered content to a safe plain-text notification body.
- [ ] Honor the user's `pm_content_in_desktop_notifications` setting for DM body visibility; use a generic “New direct message from …” form when content is disabled.
- [ ] Confirm and document channel-message body behavior.
- [ ] Never place raw HTML or untrusted URLs in title/body fields.

### Navigation

- [ ] Use existing Zulip URL/hash utilities or mirror their canonical encoding on the server.
- [ ] Build an absolute HTTPS URL under the current Zulip origin.
- [ ] For stream messages, navigate to the precise channel/topic near the message.
- [ ] For DMs, navigate to the relevant DM narrow near the message.
- [ ] Validate the final URL is same-origin before sending.
- [ ] Avoid open redirects and avoid putting secrets in the navigation URL.

### Payload size

RFC 8291 gives a maximum of approximately 3993 plaintext bytes when the push service supports only a 4096-byte body.

- [ ] Set a conservative serialized JSON cap (target 3000–3500 bytes).
- [ ] Truncate by UTF-8 bytes without splitting a code point.
- [ ] Preserve valid JSON after truncation.
- [ ] Fall back to a short generic body if required.
- [ ] Test large emoji, combining characters, long sender/channel/topic names, and long messages.
- [ ] Treat HTTP 413 as a payload-builder defect and fall back safely; do not delete the subscription.

### Deduplication/collapse

Use separate concepts:

- `notification.tag`: deterministic, privacy-safe hash based on the message/notification identity so a retry replaces the same displayed notification where supported.
- HTTP `Topic`: ≤32 URL/file-safe Base64 characters, derived from a keyed or one-way hash of the conversation identity, so an undelivered older notification for the same conversation may be replaced.

Neither value may reveal user, channel, or topic names because the HTTP `Topic` header is not encrypted.

### TTL and urgency

- [ ] Always send a `TTL` header.
- [ ] Default to a bounded chat-appropriate TTL, initially 24 hours.
- [ ] Default urgency to `normal`; RFC 8030 explicitly treats chat as a normal-urgency example.
- [ ] Do not mark every DM as high urgency.
- [ ] Revisit urgency only with a specific product requirement.

### Badges

WebKit supports an Apple-specific declarative `app_badge` member, while the cross-vendor declarative Push API specification currently defines `badge` as notification artwork rather than an application count.

- [ ] Treat `app_badge` as optional and WebKit-specific.
- [ ] Do not make v1 delivery depend on it.
- [ ] Enable it only after Phase 0/acceptance tests prove the exact payload.
- [ ] Use `navigator.setAppBadge()` / `clearAppBadge()` while the PWA is open to reconcile the icon with Zulip's unread count.
- [ ] Never let a badge mismatch block push delivery.

---

## Phase 10 — Integrate delivery with Zulip's existing worker

Reuse `missedmessage_mobile_notifications`; do not create a parallel policy queue.

### Tasks

- [ ] Query active Web Push subscriptions for the recipient after Zulip has decided a push is warranted.
- [ ] Reuse the existing late checks that suppress notifications for messages that were read, deleted, or made inaccessible before delivery.
- [ ] Build the transport-neutral notification once and dispatch it to each registered Web Push subscription.
- [ ] Keep native E2EE, APNs, FCM, and bouncer dispatch untouched except for the smallest shared integration needed.
- [ ] Count success/failure independently per transport.
- [ ] Do not let one invalid Web Push endpoint prevent native devices or another valid Web Push subscription from receiving a notification.
- [ ] Soft-reactivation behavior for personal notifications must remain consistent with existing push behavior.
- [ ] Add a user-facing test-notification route that traverses the real sender but cannot target another user's endpoint.

### Response classification

| Result | Action |
|---|---|
| 201/202 | Success |
| 404/410 | Permanently expired; delete the matching subscription |
| 429 | Transient; honor `Retry-After` within bounded queue retry policy |
| 500–599 | Transient; bounded retry |
| connect/read timeout | Transient; bounded retry |
| 400 | Payload/request bug; alert with redacted context, do not blindly purge |
| 401/403 | VAPID/configuration problem; alert, do not purge all subscriptions |
| 413 | Rebuild with generic/truncated payload; record defect |
| redirect | Reject; do not follow |

### Retry/deduplication semantics

- [ ] Define and test at-least-once behavior explicitly.
- [ ] A retry must reuse the same notification `tag`.
- [ ] Avoid retrying already successful subscriptions when practical.
- [ ] If per-subscription retries would materially complicate the initial patch, document the bounded duplicate-notification risk and rely on stable tags plus conservative retry count. Do not leave retry behavior implicit.
- [ ] Retry events must recheck message existence/read/access state before resending.

---

## Phase 11 — Handle read, delete, edit, logout, and badge semantics

A no-service-worker Declarative Web Push implementation cannot receive a silent command to remove an already displayed notification. WebKit also requires Web Push to remain user-visible.

### Required behavior

- [ ] Do not send a visible “notification removed” push.
- [ ] Continue clearing Zulip's internal `active_mobile_push_notification` state when messages are read/deleted.
- [ ] Use stable tags to reduce duplicate/retry artifacts.
- [ ] Reconcile/clear the application badge when the web app opens and when unread state changes in the foreground.
- [ ] Document that an already displayed notification may remain in Notification Center until the user opens or clears it.
- [ ] Verify message edits do not leak newly restricted/inaccessible content.
- [ ] Verify deleting a subscription or logging out stops future pushes.
- [ ] Verify logging into another account transfers or recreates the subscription without cross-account notifications.

This limitation is accepted for v1 and must be visible in operational/user documentation.

---

## Phase 12 — E2EE/privacy behavior

Zulip 12's native mobile push architecture uses application-level encryption so Apple, Google, and Zulip's bouncer cannot read message content or metadata. Standards-based Web Push uses RFC 8291 encryption from this Zulip application server to the user agent.

### Decision

Web Push subscriptions may receive notifications when the realm has `require_e2ee_push_notifications=True`, because:

- payloads are mandatorily encrypted to the browser subscription keys using `aes128gcm`;
- the hosted Zulip bouncer is bypassed;
- Apple receives the endpoint, request timing/size, VAPID routing/auth metadata, and encrypted payload, but not plaintext title/body/navigation data.

This is **not protocol-identical** to Zulip's native E2EE device protocol and must not be described as such.

### Tasks

- [ ] Add explicit tests for `require_e2ee_push_notifications=True`.
- [ ] Ensure Web Push is not classified as legacy plaintext push.
- [ ] Document residual metadata visible to Apple.
- [ ] Never log plaintext payloads in production.
- [ ] Never include subscription endpoints, `p256dh`, `auth`, VAPID private key, Authorization header, or full payload in telemetry.
- [ ] Add a security note explaining that notification content is still visible on the user's lock screen according to their iOS notification-preview settings.

---

## Phase 13 — Observability and administration

### Metrics/counters

Add low-cardinality counters through Zulip's existing analytics/logging patterns:

- active Web Push subscriptions;
- attempted deliveries;
- successful deliveries;
- expired endpoints removed;
- transient failures;
- permanent/configuration failures;
- payload fallbacks;
- test notification outcomes.

### Logging

- [ ] Identify a subscription only by a short, non-reversible endpoint hash prefix.
- [ ] Log status class and operation, not endpoint/body/keys.
- [ ] Keep stack traces free of raw request payloads and Authorization.
- [ ] Add a configuration health check or startup log indicating Web Push is enabled/disabled.
- [ ] Make test-send errors actionable for an administrator while remaining redacted.

### Cleanup

- [ ] Immediately delete 404/410 endpoints.
- [ ] Reconcile `last_seen_at` on app launch/settings.
- [ ] Add an optional management command for listing counts and pruning clearly stale rows without printing endpoints.
- [ ] Do not automatically delete merely old subscriptions unless the retention rule is documented and tested; a device may legitimately remain unopened for a long period.

---

## Phase 14 — Automated tests

### Backend unit/integration tests

- [ ] Model constraints and migration.
- [ ] Endpoint hashing and secret-redaction helpers.
- [ ] Base64URL/key validation.
- [ ] Endpoint allowlist and URL parser:
  - valid Apple endpoint;
  - suffix-confusion domains;
  - userinfo;
  - IP literal;
  - alternate port;
  - HTTP;
  - redirect;
  - malformed IDNA.
- [ ] Registration idempotency.
- [ ] Cross-account ownership transfer.
- [ ] User-scoped DELETE race.
- [ ] Per-user subscription cap.
- [ ] VAPID configured/unconfigured/malformed states.
- [ ] VAPID claim/audience/expiry construction.
- [ ] Declarative JSON schema.
- [ ] UTF-8 payload truncation and generic fallback.
- [ ] Response classification and expired endpoint deletion.
- [ ] Timeout, 429/`Retry-After`, 5xx, 401/403, 413.
- [ ] No sensitive values in emitted logs.
- [ ] Web Push-only user counts as push-registered.
- [ ] DM, mention, wildcard mention, followed topic, channel notification.
- [ ] Muted sender/channel/topic suppression.
- [ ] Online/offline setting behavior.
- [ ] Do-not-disturb behavior.
- [ ] Read-before-worker and delete-before-worker suppression.
- [ ] Message edit behavior.
- [ ] Realm configured-state lifecycle.
- [ ] `require_e2ee_push_notifications=True`.
- [ ] Native push regression cases.

Mock the remote Apple endpoint in automated tests. Never require external network access in CI.

### Frontend tests

- [ ] Feature detection.
- [ ] Normal browser tab versus installed standalone mode.
- [ ] Permission default/granted/denied.
- [ ] Enable flow and exact user-gesture entry point.
- [ ] Existing subscription reconciliation.
- [ ] Changed VAPID key resubscription.
- [ ] Registration API failure.
- [ ] Disable/logout cleanup failure modes.
- [ ] Account switch.
- [ ] Settings state rendering and translations.
- [ ] App badge feature detection/clear behavior.
- [ ] No service-worker registration is introduced.

### Required commands

At minimum:

```bash
./tools/test-js-with-node
./tools/lint
./tools/test-backend
./tools/test-migrations
```

During implementation, run focused suites first, then the complete relevant suites before merge. Provision and build frontend assets from a clean checkout.

---

## Phase 15 — Security review checklist

An independent security review is required before deployment.

- [ ] SSRF review of every path from stored endpoint to HTTP request.
- [ ] Verify redirects are disabled.
- [ ] Verify only Apple push hosts and port 443 are accepted.
- [ ] Verify no secrets enter query strings, structured logs, Sentry, analytics, or exception text.
- [ ] Verify atomic endpoint ownership transfer.
- [ ] Verify logout/account-switch isolation.
- [ ] Verify CSRF/authentication and human-user-only registration.
- [ ] Verify subscription quota/rate limiting.
- [ ] Verify payload content respects DM privacy setting.
- [ ] Verify same-origin navigation only.
- [ ] Verify HTML/script injection cannot enter notification fields.
- [ ] Verify key-file permissions and no private key in Git/build artifacts.
- [ ] Verify VAPID rotation behavior.
- [ ] Verify Web Push remains encrypted when realm requires E2EE.
- [ ] Verify disabled/malformed configuration fails closed.
- [ ] Verify the existing native push path is not weakened.
- [ ] Verify dependency license and pinned supply-chain state.
- [ ] Run repository secret scanning and dependency vulnerability scanning.

---

## Phase 16 — Manual real-device acceptance matrix

Run against a staging deployment using production-like HTTPS and a physical iPhone. Repeat the essential checks on an iPad if iPadOS is part of the supported promise.

### Installation and permission

- [ ] Safari → Share → Add to Home Screen.
- [ ] Correct icon/name and standalone launch.
- [ ] Enable control hidden or explanatory in an ordinary tab.
- [ ] Permission prompt appears only after button tap.
- [ ] Denial is handled without repeated prompting.
- [ ] Enable succeeds and test notification arrives.

### Delivery state

- [ ] PWA foreground.
- [ ] PWA backgrounded.
- [ ] PWA force-closed/terminated.
- [ ] Device locked.
- [ ] Focus mode behavior.
- [ ] Paired Apple Watch delivery, if available.
- [ ] Server/worker restart.
- [ ] Temporary outbound network failure and recovery.

### Zulip semantics

- [ ] DM.
- [ ] Personal mention.
- [ ] Group/wildcard mention.
- [ ] Followed topic.
- [ ] Channel push enabled.
- [ ] Muted topic/channel/sender.
- [ ] Do not disturb.
- [ ] “Send mobile notifications even if I'm online” on and off.
- [ ] Message read before delayed worker runs.
- [ ] Message deleted before delayed worker runs.
- [ ] DM content hidden setting.
- [ ] Notification tap opens exact message.
- [ ] Long Unicode/emoji content.
- [ ] Multiple notifications/retry deduplication.

### Device/account lifecycle

- [ ] Two iPhones registered to one user.
- [ ] Disable on one device without affecting the other.
- [ ] Logout stops future notifications.
- [ ] Login as another user on the same PWA does not leak notifications.
- [ ] Delete/reinstall PWA.
- [ ] Simulated 404/410 deletes stale server row.
- [ ] VAPID key change triggers resubscription.

### Security/configuration

- [ ] Realm requiring E2EE.
- [ ] Zulip hosted push disabled.
- [ ] Web Push-only configuration still enables mobile settings.
- [ ] Firewall blocks Apple endpoint: bounded failure, no worker hang.
- [ ] No endpoint/key/content appears in logs.

---

## Phase 17 — Deployment and rollback

### Before deployment

- [ ] Confirm production is running stock `12.2` successfully.
- [ ] Take and verify a current Zulip backup.
- [ ] Confirm the fork branch is based on the exact production release tag.
- [ ] Complete code review, security review, tests, and real-device staging acceptance.
- [ ] Generate a production VAPID key outside Git.
- [ ] Place the key under `/etc/zulip` with restrictive permissions.
- [ ] Add non-secret settings to `/etc/zulip/settings.py` and the subject/key-file path as appropriate.
- [ ] Ensure outbound TCP 443 to `*.push.apple.com`.
- [ ] Configure `/etc/zulip/zulip.conf`:

```ini
[deployment]
git_repo_url = https://github.com/LITEWERKS-GmbH/zulip.git
```

The repository is public, so a deploy key is not necessary solely for read access.

### Deploy

```bash
/home/zulip/deployments/current/scripts/upgrade-zulip-from-git litewerks-webpush
```

- [ ] Monitor `/var/log/zulip/upgrade.log`.
- [ ] Confirm migrations and frontend build complete.
- [ ] Confirm workers are healthy.
- [ ] Confirm Web Push startup health.
- [ ] Register one controlled production test device first.
- [ ] Send a test notification, then a real DM from another user.
- [ ] Expand to a small internal cohort before general instructions.

### Rollback

Zulip deployments retain `current` and `last`.

- [ ] If application behavior breaks, restart the previous deployment using its `restart-server`.
- [ ] Do not downgrade across an incompatible migration without following Zulip's database rollback guidance.
- [ ] The additive subscription table may remain harmlessly present if code is rolled back, but verify migration compatibility in staging.
- [ ] Disabling Web Push configuration must stop new Web Push delivery without affecting native push.
- [ ] Preserve the VAPID key even during rollback so existing subscriptions can be reused after a forward fix.

---

## Phase 18 — Ongoing upstream maintenance

- [ ] Keep fork `main` as a clean mirror of `zulip/zulip:main`.
- [ ] Never merge Litewerks-specific changes into `main`.
- [ ] Keep production changes on `litewerks-webpush`.
- [ ] For each reviewed stable release:
  1. fetch upstream and tags;
  2. verify release/security notes;
  3. synchronize fork `main`;
  4. rebase only the downstream commits from the old stable tag onto the new stable tag;
  5. rerun focused/full tests;
  6. repeat core iPhone acceptance;
  7. deploy the rebased branch through `upgrade-zulip-from-git`.
- [ ] Do not auto-rebase or auto-deploy production from upstream `main`.
- [ ] Monitor upstream issue `zulip/zulip#34240` and related PWA work.
- [ ] If upstream ships equivalent Web Push:
  - compare semantics/security;
  - migrate stored subscriptions if necessary;
  - remove downstream code;
  - return production to an unmodified stable release.
- [ ] Review `pywebpush`, WebKit, Push API, RFC, and dependency changes during every Zulip upgrade.
- [ ] Track the July 2026 Web Push encryption drafts as work in progress only; continue using the deployed RFC 8291 `aes128gcm` protocol until standards and Apple support change.

---

# Proposed commit/PR structure

Keep the downstream history reviewable and rebase-friendly.

1. **`webpush: add dependency and validated VAPID configuration`**
2. **`webpush: add subscription model, migration, and API`**
3. **`web: add Home Screen manifest and subscription controls`**
4. **`webpush: include web subscriptions in push eligibility/lifecycle`**
5. **`webpush: add declarative payload builder and sender`**
6. **`webpush: integrate delivery, cleanup, and badge reconciliation`**
7. **`webpush: add tests, deployment docs, and security notes`**

Implementation work should happen on a feature branch created from `litewerks-webpush`, followed by a PR back into `litewerks-webpush`. Do not code directly on fork `main`.

---

# Independent critical review

The initial high-level proposal was re-evaluated separately against current WebKit behavior, current Push API work, Zulip 12.2 internals, and operational failure modes. The following corrections and constraints are now part of the plan.

## 1. The initial effort estimate was too low

A proof of concept may be possible in 1–2 focused days. A production-ready implementation is not merely a manifest plus sender.

It must alter or audit:

- push-device eligibility before queueing;
- realm configured-state lifecycle;
- message read/delete/edit paths;
- account transfer/logout;
- endpoint cleanup/retry;
- SSRF controls;
- privacy/E2EE behavior;
- migrations, dependency provisioning, tests, and real-device rollout.

**Revised planning estimate:** approximately **7–12 focused developer-days**, excluding review scheduling and rollout soak time. This remains operationally preferable to maintaining and redistributing a native TestFlight fork, but it is a real downstream feature.

## 2. No-service-worker subscription rotation is a known gap

Declarative Web Push removes the service worker from display/delivery, but without a worker there is no `pushsubscriptionchange` handler. The mitigation is:

- reconcile `window.pushManager.getSubscription()` on every authenticated PWA launch and settings open;
- idempotently upsert;
- remove endpoints on permanent Apple responses;
- document the residual gap for devices that never reopen before rotation.

If testing shows unacceptable reliability, add the smallest possible service worker solely for subscription-change handling in a later phase. Do not add offline caching.

## 3. Web Push endpoints create an SSRF surface

This was not adequately emphasized initially. A logged-in user can supply an arbitrary URL unless constrained. For v1, Apple-only hostname allowlisting, HTTPS/443, no redirects, and strict timeout behavior are mandatory—not optional hardening.

## 4. Read/delete cannot dismiss an already displayed notification cleanly

Without a service worker, every push must remain user-visible. Sending a “remove” event would create another visible notification. v1 accepts that already displayed entries may remain in Notification Center; it clears internal state and foreground badges instead.

## 5. `app_badge` is a WebKit extension, not the same as standard `badge`

It is useful but must be feature-gated and validated on the target iOS version. Delivery cannot depend on it.

## 6. The Python sender must be proven against Apple before integration

`pywebpush` is reasonable but must be isolated, pinned, configured with explicit timeouts, and prevented from following redirects. The Phase 0 Apple transport spike is a release gate.

## 7. Account switching is a privacy boundary

Because one origin has one browser subscription, ownership must transfer atomically and logout must attempt both server deletion and local unsubscribe. Race tests are required.

## 8. VAPID key rotation invalidates existing subscriptions

Changing the application-server key requires resubscription. Rotation cannot be transparent while a PWA never opens. Preserve the key through deployments and document an explicit rotation procedure.

## 9. Web Push must enter Zulip's early eligibility query

Adding delivery only inside `handle_push_notification()` would fail: Web Push-only users are currently filtered out before the worker event is queued. The shared registered-device checks are therefore first-class scope.

## 10. Web Push encryption is strong but not identical to Zulip native E2EE

RFC 8291 protects payload plaintext from Apple and other intermediaries, satisfying the relevant privacy objective, but the implementation must not claim protocol identity with Zulip's native app E2EE. Timing, size, destination, and other routing metadata remain visible.

## 11. The patch must remain stable-based and deliberately updated

Automatically following upstream `main` would increase production risk. The fork relationship provides comparison/sync; a reviewed rebase onto stable tags preserves updateability without silently deploying upstream changes.

---

# Definition of done

This issue is complete only when all of the following are true:

- [ ] Real iOS 18.4+ Declarative Web Push works without a service worker.
- [ ] A user can install Zulip to Home Screen and explicitly enable/disable notifications.
- [ ] Web Push-only users enter Zulip's normal notification queue.
- [ ] Existing DM/mention/channel/followed-topic, mute, DND, and online/offline semantics are preserved.
- [ ] Notification taps navigate to the correct same-origin message.
- [ ] Multiple devices and account switches are safe.
- [ ] Expired endpoints are removed and transient errors are bounded.
- [ ] SSRF protections, secret redaction, quotas, and rate limits are tested.
- [ ] Standards-based payload encryption works when realm E2EE is required.
- [ ] No native APNs/FCM/bouncer behavior regresses.
- [ ] Backend/frontend/migration/lint test suites pass.
- [ ] An independent security review has no unresolved high/critical findings.
- [ ] Staging passes the physical-device acceptance matrix.
- [ ] Production deployment and rollback instructions are validated.
- [ ] No VAPID private key or production subscription secret exists in Git.
- [ ] The downstream commits remain cleanly separable from upstream Zulip.
- [ ] Documentation explains the no-service-worker rotation gap and notification-dismissal limitation.

---

# Primary references

## Apple/WebKit

- Declarative Web Push design and `window.pushManager`:  
  https://webkit.org/blog/16535/meet-declarative-web-push/
- Safari/WebKit 18.4 release support:  
  https://webkit.org/blog/16574/webkit-features-in-safari-18-4/
- Original iOS/iPadOS Home Screen Web Push behavior and Apple network requirements:  
  https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/
- Home Screen web-app badging:  
  https://webkit.org/blog/14112/badging-for-home-screen-web-apps/

## Standards

- W3C Push API, including Declarative Push Message:  
  https://w3c.github.io/push-api/
- Known no-service-worker subscription-change gap:  
  https://github.com/w3c/push-api/issues/401
- RFC 8030 — Generic Event Delivery Using HTTP Push:  
  https://datatracker.ietf.org/doc/html/rfc8030
- RFC 8291 — Message Encryption for Web Push:  
  https://datatracker.ietf.org/doc/html/rfc8291
- RFC 8292 — Voluntary Application Server Identification (VAPID):  
  https://datatracker.ietf.org/doc/html/rfc8292
- Web App Manifest:  
  https://www.w3.org/TR/appmanifest/

## Zulip

- Upstream PWA/Web Push feature request:  
  https://github.com/zulip/zulip/issues/34240
- Closed upstream manifest attempt and review history:  
  https://github.com/zulip/zulip/pull/33592
- Notification subsystem documentation:  
  https://github.com/zulip/zulip/blob/12.2/docs/subsystems/notifications.md
- Production modifications/upgrades:  
  https://github.com/zulip/zulip/blob/12.2/docs/production/modify.md
- Mobile push security/help:  
  https://github.com/zulip/zulip/blob/12.2/starlight_help/src/content/docs/mobile-notifications.mdx
- Dependency management:  
  https://github.com/zulip/zulip/blob/12.2/docs/subsystems/dependencies.md

## Sender dependency candidate

- `pywebpush`:  
  https://github.com/web-push-libs/pywebpush
