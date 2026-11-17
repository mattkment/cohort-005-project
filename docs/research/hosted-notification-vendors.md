# Research: hosted notification vendors

Ticket: #2 (map #1). Researched 2026-11-17. Facts only; no vendor is chosen here.

Sources are official docs and pricing pages. Pages were read via a summarizing fetch tool, so exact figures should be re-checked on the pricing page before committing money. Items marked **UNVERIFIED** were not confirmed from a primary page.

Cadence stack: React Router 8 (SSR), React 19, SQLite/Drizzle, Node. Existing email: Postmark in `app/lib/mailer.server.ts`.

## Summary table

| | Knock | Novu Cloud | Courier | MagicBell | OneSignal |
|---|---|---|---|---|---|
| In-app inbox, React | Yes: prebuilt + hooks | Yes: prebuilt + hooks | Yes: prebuilt + custom | Yes (React components + headless) | **No inbox/feed found on pricing page** |
| Browser web push | Not documented (push page lists mobile providers, OneSignal) | Via Pushpad / FCM / Push Webhook providers | Not documented in pages read (mobile APNs/FCM) | Listed ("Web Push") | Yes, native (service worker) |
| Email via Postmark | Yes (channel provider) | Yes (provider) | Yes (provider) | Not listed (SES, Mailgun, Ping, SMTP, SendGrid) | Not checked |
| Free tier | 10,000 messages/mo | 10,000 workflow runs/mo | 10,000 sends/mo | 1,000 deliveries/mo | 10,000 emails/mo; in-app + mobile push unlimited up to 1,000 MAU |
| First paid tier | $250/mo | $30/mo | $0.005/send, no base | $249/mo | Growth from $19/mo |

## Knock

- **In-app feed.** `@knocklabs/react` ships `NotificationFeedPopover`, `NotificationFeed`, `NotificationIconButton` (bell with count) and a set of hooks for a custom UI. Providers: `KnockProvider`, `KnockFeedProvider`. [React overview](https://docs.knock.app/in-app-ui/react/overview), [React feed](https://docs.knock.app/in-app-ui/react/feed)
- **Counts / read / archive.** Badge uses seen counts by default; `badgeCountType="unread"` switches to unread. Popover marks items seen on open; mark-read via `feedClient.markAsRead`. The JS feed client exposes archive (`markAsArchived`) and `useNotifications` returns the feed client for read/seen/archived and realtime. [use-notifications](https://docs.knock.app/in-app-ui/react/sdk/hooks/use-notifications), [feed client](https://docs.knock.app/in-app-ui/javascript/sdk/feed-client)
- **Client auth.** Feeds are open to anyone with the feed ID by default. Enabling enhanced security in the dashboard requires a signed `userToken` passed to `KnockFeedProvider`. [React feed](https://docs.knock.app/in-app-ui/react/feed)
- **Triggering.** Workflows run via API trigger endpoint (also source events, schedules, audiences). Node: `@knocklabs/node`, `workflows.trigger(key, {...})`, `users.identify(id, {...})`; recipients are user IDs or inline identities, up to 1,000 per trigger. [Triggering workflows](https://docs.knock.app/send-notifications/triggering-workflows/overview), [API trigger](https://docs.knock.app/send-notifications/triggering-workflows/api), [knock-node](https://github.com/knocklabs/knock-node). Exact current Node method signature not confirmed from a primary page.
- **Email.** Postmark is a supported email channel provider (API key + from address; optional delivery webhooks). So Knock can send via the existing Postmark account. [Postmark integration](https://docs.knock.app/integrations/email/postmark)
- **Push.** Push docs list Amazon SNS, APNs, Expo, FCM (labelled Android), OneSignal. Browser web push is not mentioned. [Push overview](https://docs.knock.app/integrations/push/overview)
- **Pricing.** Free $0: 10,000 messages/mo, 30-day logs, Knock branding stays, 99.9% SLA. Starter $250/mo: 50,000 messages then $0.005 each. Unlimited profiles, workflows, channels, team members on all plans. [Pricing](https://knock.app/pricing)
- **Unknown from sources read:** React version support, SSR guidance.

## Novu (cloud)

- **In-app inbox.** `@novu/react` `<Inbox />` (Bell, Notifications, InboxContent, Preferences parts) or headless hooks / `@novu/js`. Unread count on the bell updates in real time; read state syncs to backend. Requires React 18+; React 19 supported. No SSR-specific guidance (links a Next.js quickstart). Archive not mentioned on the pages read. [Inbox overview](https://docs.novu.co/platform/inbox/overview)
- **Identity / security.** Subscriber ID plus application identifier. Production: enable HMAC on the Novu In-App integration; server computes `HMAC-SHA256(subscriberId, secretKey)` hex and passes it as `subscriberHash`. [Prepare for production](https://docs.novu.co/platform/inbox/prepare-for-production)
- **Triggering.** `POST /v1/events/trigger` or `@novu/api`: `novu.trigger({ workflowId, to: subscriberId, payload })`; up to 100 recipients; `Idempotency-Key` header supported but must be enabled per organization by support. [Trigger event](https://docs.novu.co/api-reference/events/trigger-event)
- **Email.** Postmark is a supported provider (server API token + verified sender). [Postmark](https://docs.novu.co/platform/integrations/email/postmark)
- **Push.** Providers listed in search results: FCM, Expo, APNs, OneSignal, Pushpad, Push Webhook, Pusher Beams. Pushpad is described as browser web push (needs Pushpad SDK on the site and a uid per subscription); FCM web has a (v0.x) guide. [Push integrations](https://docs.novu.co/platform/integrations/push), [Pushpad](https://docs.novu.co/platform/integrations/push/pushpad)
- **Pricing.** Free $0: 10,000 workflow runs/mo, 5 workflows, 2 environments, 3 team members, 24h activity-feed retention, unlimited subscribers, all channels. Pro from $30/mo: 30,000+ runs, 20 workflows, 7-day retention, removes branding, overage $1.20 per 1K runs. Team from $250/mo. [Pricing](https://novu.co/pricing)
- **Self-host.** Enterprise plan lists self-hosted deployment; irrelevant here as cloud only is required.

## Courier

- **In-app inbox.** `@trycourier/courier-react` v9 (React 18+; separate `courier-react-17` package). Out-of-the-box notification center, themeable via `CourierInboxTheme`, or custom UI on low-level APIs. Unread counts/badges, read and archive actions, infinite scroll; state syncs across devices in real time. [Inbox overview](https://www.courier.com/docs/platform/inbox/inbox-overview)
- **Client auth.** Server mints a JWT via `POST /auth/issue-token` with scopes `user_id:<id> inbox:read:messages inbox:write:events`; client calls `signIn({ userId, jwt })`. The user need not exist in Courier first. Omitting `expires_in` makes a non-expiring token; SDKs do not auto-refresh. [Authentication](https://www.courier.com/docs/platform/inbox/authentication)
- **Triggering.** Send API; Node SDK `@trycourier/courier` (Node 20+) `client.send.message` with `to: { user_id }`, a template with an In-App channel, `routing.channels: ["inbox"]`. Inbox requires `user_id`. [Send to the inbox](https://www.courier.com/docs/in-app/send-to-the-inbox.md), [Node quickstart](https://www.courier.com/docs/quickstarts/node.md)
- **Email.** Postmark supported ("Keep Postmark as your email provider"); per-message overrides incl. MessageStream and Postmark templates. [Postmark](https://www.courier.com/docs/external-integrations/email/postmark)
- **Push.** Docs read describe mobile push (APNs/FCM). Browser web push not confirmed. [Inbox overview](https://www.courier.com/docs/platform/inbox/inbox-overview)
- **Pricing.** Developer free: 10,000 sends/mo. Business: $0.005 per send, billed from the first send. Enterprise custom. The pricing page does not list Inbox by plan, so plan availability of Inbox is **UNVERIFIED**. [Pricing](https://www.courier.com/pricing)

## MagicBell

Less verified: several doc URLs returned 404 and only the docs landing page and search snippets were read.

- Positions itself as "The Notification Inbox for web & mobile apps". SDKs: React components, headless React, Preact, Svelte, Web Components, Node/browser clients. Channels listed: Web Push, In-App, mobile push, Slack, SMS, Teams, Email (SES, Mailgun, Ping Email, SMTP, SendGrid). **Postmark is not listed.** Auth: Project JWT and User JWT listed. [Docs](https://www.magicbell.com/docs)
- Headless store methods (v5.x type definitions, via search) include mark seen/read/unread, archive, delete. React 19 support, SSR, and trigger API shape **UNVERIFIED**. [React headless](https://magicbell.com/docs/libraries/react-headless)
- **Pricing.** Builder free: 1,000 deliveries/mo, 1 project. Startup $249/mo: 50,000 deliveries, then $0.0025 each. [Pricing](https://www.magicbell.com/pricing)

## OneSignal

- **Positioning.** Push/email/in-app messaging platform, not an inbox-first product. The pricing page lists In-App Messaging but no notification inbox or feed; no web inbox component was found in the pages read. **UNVERIFIED** whether one exists elsewhere in the docs. [Pricing](https://onesignal.com/pricing)
- **Web push.** Requires HTTPS, a service worker served from the same origin, and per-origin app; users must grant permission; subscriptions stay anonymous until an External ID is set via login. [Web push setup](https://documentation.onesignal.com/docs/en/web-push-setup)
- **Identity.** JWT identity verification (ES256, claims `iss`, `exp`, `identity.external_id`). Per OneSignal docs, web SDK cannot send the JWT yet, so enabling the app-wide toggle breaks web logins. [Identity verification](https://documentation.onesignal.com/docs/en/identity-verification)
- **Pricing.** Free: 10,000 email sends/mo; in-app messaging and mobile push unlimited up to 1,000 MAU; web push cap unclear on the page. Growth from $19/mo. [Pricing](https://onesignal.com/pricing)
- Knock and Novu both list OneSignal as a push provider, so it can sit behind them as a web push delivery channel.

## Cross-cutting facts

- All four inbox vendors (Knock, Novu, Courier, MagicBell) offer a React feed with prebuilt and headless options; only Novu's page confirmed React 19.
- Knock, Novu and Courier can send email through Postmark, so they can augment or route via the existing account rather than require replacing it. MagicBell does not list Postmark.
- Browser web push: clearly supported natively by OneSignal and listed by MagicBell; reachable in Novu via Pushpad/FCM; not confirmed for Knock or Courier from pages read (Knock can use OneSignal as a push provider).
- Client identity patterns: Knock signed `userToken`; Novu HMAC `subscriberHash`; Courier server-minted JWT with scopes; MagicBell JWT; OneSignal External ID (+ JWT, not web-capable yet).
- No candidate was disqualified on the criteria in the ticket from what was read. OneSignal lacks a confirmed inbox/feed.
- Free-tier retention: Novu free keeps activity feed only 24 hours (activity feed is the Novu dashboard log; whether this affects the end-user inbox list is **UNVERIFIED**). Knock free logs 30 days.

## Gaps to close if needed

- React 19 / React Router SSR compatibility statements for Knock, Courier, MagicBell.
- Inbox availability per Courier plan; MagicBell docs detail; OneSignal inbox existence.
- Web push specifics for Knock and Courier; message retention for end-user inbox items on free tiers.
- Confirm current Knock Node `workflows.trigger` signature from the SDK README/types.
