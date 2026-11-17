# Research: is browser push viable without a mobile app?

Ticket: #3 (parent map #1). Facts only; the channel-strategy decision is a separate ticket.
Researched 2026-11-17. Each claim cites its source. "Not confirmed" marks things the fetched primary source did not state.

## 1. Browser support

MDN browser-compat-data for `PushManager` (https://raw.githubusercontent.com/mdn/browser-compat-data/main/api/PushManager.json):

| Browser | Supported since | Note in source |
| --- | --- | --- |
| Chrome (desktop) | 42 | |
| Chrome Android | mirrors Chrome | |
| Edge | 17 | |
| Firefox (desktop) | 44 | |
| Firefox Android | 48 | |
| Safari (macOS) | 16 | "Notifications are supported on macOS Ventura and later." |
| Safari iOS/iPadOS | 16.4 | "Notifications are supported in web apps saved to the home screen." |

- MDN lists the Push API as Baseline "Widely available", available across browsers since March 2023 (https://developer.mozilla.org/en-US/docs/Web/API/Push_API).
- WebKit: standards-based Web Push arrived in Safari 16.1 for macOS Ventura; iOS/iPadOS 16.4 added it for Home Screen web apps (https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/). (MDN data says Safari 16; WebKit's post says 16.1. Minor discrepancy between sources.)

### iOS Home Screen requirement

From WebKit (https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/):
- Web Push on iOS/iPadOS applies only to Home Screen web apps. The site needs a web app manifest with `display` of `standalone` or `fullscreen`, and the user adds it via Share > "Add to Home Screen".
- The permission request must be "in response to direct user interaction" (e.g. a subscribe button); iOS then shows its own prompt.
- Delivery goes through APNs; no Apple Developer Program membership needed. Sites that control push endpoints should allow `*.push.apple.com`.
- Users manage permission per web app in Notification Settings; notifications appear on Lock Screen, Notification Center, and a paired Apple Watch. They integrate with Focus.
- OneSignal docs confirm the practical effect: on iOS 16.4+ "users must add your site to their Home Screen. iOS doesn't show the permission prompt until they do" (https://documentation.onesignal.com/docs/web-push-setup).
- Not confirmed from fetched sources: behavior in a regular Safari tab on iOS (WebKit's post does not address it; the MDN note and vendor docs imply it is not supported there). Apple's developer documentation page could not be retrieved in usable form.

## 2. Permission model and opt-in friction

- `Notification.permission` is `default` (not asked), `granted`, or `denied` (https://developer.mozilla.org/en-US/docs/Web/API/Notifications_API/Using_the_Notifications_API).
- Permission should be requested only from a user gesture; Firefox enforces this since v72 and Safari has for some time (same MDN page). Chrome and Firefox require a secure context (HTTPS). Cross-origin iframes can't be granted permission (same page).
- Chrome's Lighthouse guidance flags calling `requestPermission()` on page load and recommends offering a specific notification type first, then requesting permission after opt-in (https://developer.chrome.com/docs/lighthouse/best-practices/notification-on-start).
- What users see: a browser-native prompt with Allow / Block, or they can close it (https://web.dev/articles/push-notifications-subscribing-a-user). If blocked, "the user must manually unblock" via browser settings; the page does not state whether a dismissed prompt can be re-asked.
- web.dev: "Most users grant permission if they understand why the app requests it." (same article).
- OneSignal documents a pre-prompt pattern (custom in-page prompt before the native dialog) and notes private/incognito browsing can't subscribe (https://documentation.onesignal.com/docs/web-push-setup).
- Opt-in is per browser and device: a subscription is created per browser/device combination where permission is granted (OneSignal doc above).
- Not confirmed from fetched sources: Firefox/Chrome "quieter" prompt UI for low-engagement sites, and opt-in rate figures. No primary data on typical opt-in percentages was gathered.

## 3. What a site must build (self-built)

Sources: MDN Push API, js13kGames push tutorial, web.dev Web Push Protocol.
- A **service worker** receives push messages via the `push` event and calls `showNotification()` (https://developer.mozilla.org/en-US/docs/Web/API/Push_API; https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Tutorials/js13kGames/Re-engageable_Notifications_Push).
- The page calls `PushManager.subscribe({ userVisibleOnly: true, applicationServerKey })`. The result is a `PushSubscription` with an `endpoint` and keys (`p256dh`, `auth`).
- **VAPID keys**: the app holds a key pair; the public key is the `applicationServerKey`; the server signs a JWT with the private key (claims `aud`, `exp` max 24h, `sub` contact URL/mailto) (https://web.dev/articles/push-notifications-web-push-protocol).
- **Subscription storage**: the server must store each subscription's endpoint and keys to send later. Endpoints are capability URLs and must be kept secret; subscriptions must be protected against CSRF (MDN Push API). The `auth` value is a secret (web.dev protocol article).
- **Sending**: the app server encrypts the payload (ECDH + HKDF + AES-GCM), signs the VAPID JWT, and POSTs to the endpoint; the push service delivers it. Required `TTL` header; optional `Topic`, `Urgency`. Response 404/410 means the subscription is gone and should be deleted; 413 payload too large (min 4096 bytes supported); 429 rate limited (web.dev protocol article). The Node `web-push` library handles key generation and sending (js13kGames tutorial).
- Every push must yield a visible notification when `userVisibleOnly: true` is used (js13kGames tutorial). Firefox limits non-notification pushes per visit; Chrome has no limit (MDN Push API).
- `onpushsubscriptionchange` fires when a subscription is invalidated or about to expire, so resubscription logic is needed (MDN Push API).
- Apple delivers via APNs but the app server still posts to the standard endpoint (WebKit post); all push services share the same Web Push API (web.dev protocol article).
- A web app manifest is additionally needed for iOS (WebKit post).

## 4. What a hosted vendor handles

- **OneSignal** (https://documentation.onesignal.com/docs/web-push-setup): browsers supported are Chrome, Firefox, Safari (macOS and iOS 16.4+), Edge. Stores subscriptions per browser/device; anonymous until an External ID is assigned; optional auto-resubscribe; configurable prompt types (slidedown, native, etc.). The site must still host the OneSignal service worker on its own origin. The page does not cover VAPID handling. iOS still needs a manifest and Home Screen install.
- **Firebase Cloud Messaging (web)** (https://firebase.google.com/docs/cloud-messaging/js/client): requires HTTPS, a `firebase-messaging-sw.js` in the domain root, a VAPID key pair generated in the Firebase console, and `Notification.requestPermission()`. Works in browsers supporting the Push API. The page does not address Safari/iOS web. The registration-token API is described as deprecated in favour of the Firebase Installation ID.
- **MagicBell** (https://www.magicbell.com/docs): lists both "Notification Inbox" and "Web Push Notifications" products, with Web Push and In-App channel docs.
- **Novu** (https://docs.novu.co/platform/integrations/push): push providers listed are FCM, Expo, APNS, OneSignal, Push Webhook, Pusher Beams, Pushpad. Web push is not a standalone provider; docs mention using FCM for a web app.
- **Knock** (https://docs.knock.app/integrations/push/overview): push providers listed are Amazon SNS, APNs, Expo, FCM, OneSignal. The page does not mention browser web push or the in-app feed (Knock's feed is documented elsewhere; not fetched).
- **Courier** (https://www.courier.com/docs/external-integrations/push/intro): push providers listed are APNs, FCM, Expo, OneSignal, Airship, Amazon SNS, Pusher Beams, Pusher, MagicBell, Pushbullet, Beamer, NowPush. Browser web push not mentioned on that page.
- Pattern: several workflow vendors (Novu, Knock, Courier) route push through a downstream push provider (e.g. OneSignal, FCM) rather than owning the browser subscription flow themselves. Vendors differ on whether the service worker/prompt UX is owned by the vendor SDK or by the site.
- Not verified: pricing, current feature sets beyond the cited pages, and web push support for vendors not listed.

## 5. Practical findings (facts, not a decision)

- Reach: all current desktop browsers (Chrome, Edge, Firefox, Safari 16+) and Android browsers support Web Push. On iOS it works only for sites installed to the Home Screen as web apps (iOS 16.4+), so a user on iPhone must first add Cadence to the Home Screen before any prompt can appear.
- Opt-in is per device/browser, requires an explicit user gesture and native permission prompt, and a blocked state can only be reversed in browser settings.
- Self-building requires a service worker, VAPID keys, secure subscription storage, encryption/sending, and expired-subscription cleanup; a vendor can handle storage, sending and prompt UX, but the site still hosts a service worker.
- Email in Cadence already exists (Postmark, `app/lib/mailer.server.ts`, per map #1) and needs no per-device permission prompt; the sources above contain no email-vs-push comparison data, so any comparison of delivery or engagement is out of scope of these facts.

## Gaps to verify if needed

- Apple developer doc page for Web Push (fetch returned no content) and exact Safari-tab-on-iOS behavior.
- Chrome/Firefox quiet-permission UI behavior and Mozilla's support article (fetch failed).
- Knock and Courier web push/feed support via other doc pages; Novu inbox details; pricing for all vendors.
