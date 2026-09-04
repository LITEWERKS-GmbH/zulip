# Litewerks Web Push implementation scope

## Goal

Provide reliable push notifications for the LITEWERKS self-hosted Zulip installation on iPhone without Zulip's hosted mobile push service or a custom native app.

Initial supported client: iOS 18.4+ Home Screen web app using Declarative Web Push.

## Architectural constraints

- Reuse Zulip's existing notification eligibility and missed-message queueing logic.
- Add Web Push as a delivery transport; do not fork/reimplement notification policy.
- Keep the patch isolated enough to rebase cleanly onto future stable Zulip tags.
- Keep `main` pristine and upstream-compatible.
- No Apple Developer account, APNs credential, Firebase configuration, or native mobile fork should be required.
- Never commit VAPID private keys or production secrets.

## Implementation slices

### 1. Installability

- Add/complete standards-compliant web app manifest metadata.
- Reuse appropriate existing Zulip icons/assets where possible.
- Ensure the installed Home Screen app opens in standalone mode and routes correctly to the realm.
- Do not introduce offline caching of Zulip content as part of this work.

### 2. Subscription UX

- Detect supported iOS/Declarative Web Push capability.
- Add an explicit user-gesture-driven action to enable notifications.
- Request notification permission only from that user action.
- Create a PushManager subscription using the server's public VAPID key.
- Register/update the subscription with the authenticated Zulip account.
- Provide disable/unsubscribe behavior.
- Fail safely and visibly if permission is denied or subscription fails.

### 3. Server persistence/API

- Add a dedicated Web Push subscription model associated with a user.
- Store endpoint and Web Push key material required by the standard (`p256dh`, `auth`, content encoding/capability as required).
- Support multiple installed devices/subscriptions per user.
- Provide authenticated register/update/delete endpoints.
- Add migration and tests.

### 4. VAPID configuration

- Add server configuration for VAPID public/private key material.
- Read the private key from Zulip's server-side secrets/configuration mechanism under `/etc/zulip`.
- Expose only the public key to the web client.
- Push must remain disabled when configuration is incomplete rather than failing the notification worker.

### 5. Delivery integration

- Hook Web Push delivery into the existing missed-message/mobile-notification worker after Zulip has already decided the user should be notified.
- Do not modify the semantics of mentions, DMs, channel notifications, muted topics, idle handling, or existing notification preferences unless required for correct integration.
- Use standard Web Push encryption/VAPID.
- For iOS 18.4+, emit a Declarative Web Push payload with title/body, navigation target, and badge where supported.
- Ensure sending to Web Push does not require Zulip's hosted notification bouncer.

### 6. Navigation and lifecycle

- Notification tap must open/focus the installed Zulip web app and navigate to the intended message/conversation.
- Handle expired/invalid Web Push endpoints by removing or disabling stale subscriptions.
- Avoid uncontrolled duplicate notifications when the same subscription is registered repeatedly.

### 7. Tests

At minimum cover:

- subscription create/update/delete authorization and validation;
- multiple subscriptions per user;
- VAPID/configuration-disabled behavior;
- sender payload construction;
- expired subscription cleanup;
- integration with existing notification eligibility rather than bypassing it;
- stable deep-link construction.

## Manual acceptance criteria

On a real iPhone running iOS 18.4+:

1. Open the production/staging Zulip URL in Safari.
2. Add it to the Home Screen.
3. Open the Home Screen app and enable notifications from the in-app action.
4. Fully leave/close the web app.
5. Send a Zulip DM/mention that should generate a mobile notification.
6. A lock-screen/Notification Center notification arrives without the official Zulip native app being installed or Zulip's hosted push service being enabled.
7. Tapping the notification opens the installed Zulip web app at the correct conversation/message context.
8. Muted/non-notifiable messages follow existing Zulip notification policy.
9. Disabling notifications/unsubscribing stops future Web Push delivery to that installation.
10. Re-login/restart and normal Zulip browsing remain unaffected.

## Deployment gate

Do not deploy the branch to production until:

- the production server has completed its normal upgrade to the same stable baseline used by this branch;
- relevant Zulip automated tests pass;
- Web Push tests pass;
- manual staging acceptance succeeds on a real iPhone;
- `/etc/zulip` VAPID configuration is prepared separately from Git;
- rollback to the previous `/home/zulip/deployments/last` deployment is understood/available.
