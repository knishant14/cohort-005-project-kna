# In-app notification providers — comparison

Research for issue #2 (part of #1). Facts only, no recommendation. Scope: in-app inbox only (no email/push). Retrieved 2026-11-10 from first-party docs, pricing pages, GitHub and the npm registry. Pricing changes often; re-check before deciding.

Candidates: Novu, Knock, MagicBell, plus Courier and SuprSend (the other hosted notification platforms with an in-app feed that appeared in first-party pricing pages). Building our own inbox in SQLite/Drizzle is the implicit baseline and is not researched here.

## Summary table

| | Novu | Knock | MagicBell | Courier | SuprSend |
|---|---|---|---|---|---|
| Free tier | 10,000 workflow runs/mo, 5 workflows, 2 envs, 3 team members, 24h activity-feed retention | 10,000 messages/mo, 500 "guide recipients"/mo, unlimited workflows/profiles/members | 1,000 deliveries/mo, 1 project, unlimited users and team members | 10,000 sends/mo | 10,000 notifications/mo; in-app feed carries SuprSend branding |
| First paid | Pro from $30/mo (30K runs) | Starter $250/mo (50K messages) | Startup $249/mo (50K deliveries) | Business $0.005/send, pay-as-you-go | Essentials $100/mo billed annually (50K) |
| Self-host | Yes (community edition, MongoDB + Redis + S3); Enterprise self-host option on pricing page | No self-hosting found | No self-hosting found | No self-hosting found | Enterprise only ("Self-hosted deployment", BYOC beta) |
| React package | `@novu/react` 3.19.2 | `@knocklabs/react` 0.15.1 | `@magicbell/react` 1.14.0, `@magicbell/react-headless` 5.5.1 | not examined | not examined |

## Novu

- **Pricing / free tier** — Free $0: 10K workflow runs/mo, all channels, 5 workflows, 2 environments (Dev + Prod), 24h activity-feed retention, up to 3 team members, unlimited subscribers. Pro from $30/mo (30K runs, $1.20 per extra 1K, 7-day retention, removes Novu branding). Team from $250/mo. Enterprise custom, includes self-hosted option. Source: https://www.novu.co/pricing/
- **React SDK / React Router v7 fit** — `@novu/react` provides `<Inbox />`, plus headless mode and hooks (`useNovu`, `useCounts`, `usePreferences`). Peer deps: React `^18 || ^19`, react-dom. The official React quickstart is "add routing with React Router": `<Inbox routerPush={(path) => navigate(path)} />` using `useNavigate`. Remix quickstart also exists (`@novu/react`, `<Inbox />` placed in a component). No React Router v7-specific page found; the `routerPush` prop is the integration point. Sources: https://docs.novu.co/platform/inbox/setup-inbox.md, https://docs.novu.co/platform/quickstart/react.md, https://docs.novu.co/platform/quickstart/remix.md, https://registry.npmjs.org/@novu/react/latest
- **Server-side trigger API** — `POST /v1/events/trigger` (`https://api.novu.co` or `https://eu.api.novu.co`), secret-key auth, workflow identifier + subscriber(s) + payload, max 100 recipients per call, returns a `transactionId`. `Idempotency-Key` header dedupes retries (not enabled for all orgs; contact support). Server SDKs: Node, Python, Go, PHP, .NET, Java. Sources: https://docs.novu.co/api-reference/events/trigger-event, https://docs.novu.co/platform/sdks.md
- **Subscriber model** — A subscriber is identified by an app-defined, stable `subscriberId` (docs recommend the internal user ID). Profile holds `email`, `phone`, `firstName`, `locale`, a free-form `data` object, and channel credentials. Subscribers are created ahead of trigger or just-in-time on trigger. Source: https://docs.novu.co/platform/concepts/subscribers.md
- **Read/unread** — Client API: `novu.notifications.read/unread/archive/readAll/archiveAllRead`; `useCounts` exposes total, unread and unseen counts in real time. Sources: https://docs.novu.co/platform/sdks/react/hooks/use-novu.md, https://docs.novu.co/platform/sdks/react/hooks/use-counts.md (via index)
- **Preferences** — Layered: workflow channel preferences (code/dashboard) → subscriber global → subscriber per-workflow; most specific wins; "critical" workflows ignore subscriber choices. Built-in Preferences view in `<Inbox />`, `<Subscription />` component, `usePreferences` hook, and preferences APIs. Stored per environment (and per tenant when a tenant context is used). Source: https://docs.novu.co/platform/concepts/preferences.md
- **Client auth** — `applicationIdentifier` + `subscriberId` in the browser; production should enable HMAC: server computes `subscriberHash` from the secret key and passes it to the Inbox. Without it, existing subscriber profile fields are not updated from the client. Source: https://docs.novu.co/platform/inbox/prepare-for-production.md
- **Local dev / testing** — Two environments on Free (Dev + Prod), more on higher plans. Local workflow development is via the Framework / dashboard. A local-only (offline) mode was not verified in this research.
- **Self-hosting** — Open source on GitHub (repo license reported by GitHub as "Other": MIT plus `LICENSE-ENTERPRISE`/`EE-PACKAGES-LICENSE` for enterprise packages); `docker/community` has a docker-compose. Requires MongoDB, Redis (AOF on queue instance) and S3-compatible storage; single-VM spec 4 vCPU/8 GB. Some cloud-only features are unavailable when self-hosted; no social login. **Not SQLite-compatible**: its datastore is MongoDB. Sources: https://docs.novu.co/community/self-hosting-novu/overview.md, https://github.com/novuhq/novu, https://docs.novu.co/platform.md

## Knock

- **Pricing / free tier** — Free $0: 10,000 messages/mo (one message = one delivery to one user on one channel), 500 guide recipients, unlimited workflows, profiles, broadcasts, channels and team members, built-in multi-tenancy, end-to-end testing, preferences. Starter $250/mo (50K messages then $0.005 each; removes Knock branding). Enterprise custom. Dormant profiles are not billed. Source: https://knock.app/pricing
- **React SDK / React Router v7 fit** — `@knocklabs/react`: `KnockProvider`, `KnockFeedProvider`, `NotificationFeed`, `NotificationFeedPopover`, `NotificationIconButton`, toasts, preference center, plus headless hooks. Peer deps: React 17/18/19; `next` and `@tanstack/react-router` are optional peers. No React Router–specific integration; click handling is via a callback ("handling interactivity"). Quickstart is Vite React using `VITE_*` env vars. Sources: https://docs.knock.app/in-app-ui/react/overview, https://docs.knock.app/getting-started/quick-start/react.md, https://registry.npmjs.org/@knocklabs/react/latest
- **Server-side trigger API** — `workflows.trigger`: `key`, `recipients` (user IDs, objects, or inline-identified), optional `actor`, `data` (10 MB limit), `cancellation_key`, `tenant`; returns `workflow_run_id`. Recipients can be inline-identified in the trigger call. OpenAPI spec published. Source: https://docs.knock.app/send-notifications/triggering-workflows/api.md
- **Subscriber (user) model** — Users keyed by your own `id` (max 256 chars, no `/` or `#`, immutable once set). Also supports non-user recipients (Objects) and subscriptions. Users must be identified (synced) to Knock or inline-identified on trigger/client init. Source: https://docs.knock.app/concepts/users.md
- **Read/unread** — Feed API returns messages newest-first with unread/read/seen counts, filterable by status (unread, read, seen, archived) or custom data; engagement statuses (seen, read, interacted, archived) are updated by the client. Real-time delivery by managed websocket. In-app message data and the Feeds API are not subject to log retention enforcement. Sources: https://docs.knock.app/in-app-ui/feeds/overview.md, https://docs.knock.app/concepts/messages.md, https://docs.knock.app/send-notifications/message-statuses.md
- **Preferences** — Per-user `PreferenceSet` (channels, categories, workflows); evaluated at workflow run; hosted preference center, React preference center component and headless hooks. Source: https://docs.knock.app/concepts/preferences.md
- **Client auth** — Public API key in browser; "enhanced security mode" (signed user token, optional grants) recommended for production. Source: https://docs.knock.app/in-app-ui/security-and-authentication.md
- **Local dev / testing** — Dashboard workflow test runner, workflow debugger, environments (production by default; enable development + more), each with own API keys and isolated users/messages. Knock CLI and Management API for managing resources from code. No local emulator found. Sources: https://docs.knock.app/send-notifications/testing-workflows.md, https://docs.knock.app/version-control/environments.md
- **Self-hosting** — None found in docs or pricing. SaaS only.

## MagicBell

- **Pricing / free tier** — Builder $0: 1,000 deliveries/mo, 1 project, unlimited users and team members, all integrations. Startup $249/mo: 50K deliveries, $0.0025 each extra, 5 projects, $99 per extra project. Enterprise custom. A delivery is counted per user per channel. Overage is billed, not blocked. Source: https://www.magicbell.com/pricing
- **React SDK / React Router v7 fit** — `@magicbell/react` (1.14.0) and `@magicbell/react-headless` (5.5.1, React ≥ 18.3.1). Quickstart is an embed code from the dashboard with theming via CSS; no framework-specific or router-specific docs found. The npm `@magicbell/react` package declares no peer dependencies. Sources: https://www.magicbell.com/docs/quick-start, https://registry.npmjs.org/@magicbell/react-headless/latest
- **Server-side trigger API** — REST v2 `POST /broadcasts` at `https://api.magicbell.com/v2` with project JWT auth. A broadcast has title/content, `category`, `action_url`, `custom_attributes`, per-channel overrides (including `in_app`), recipients (users or topics); one notification is created per recipient. SDK: `magicbell-js`. Sources: https://www.magicbell.com/docs/api/reference, https://www.magicbell.com/docs/primitive/broadcast
- **User model** — Identified by `email` or `external_id` (docs prefer `external_id`; immutable once set; external ID wins over email). Users are upserted and auto-created when the Inbox initializes with a user JWT. Source: https://www.magicbell.com/docs/primitive/user
- **Read/unread** — Notifications track read, seen and archived state with `sent_at`/`read_at`/`seen_at` timestamps. Source: https://www.magicbell.com/docs/primitive/notification
- **Preferences** — Users manage delivery preferences; categories and topics are the unit ("use categories and topics … for better user preference management"). Detailed preferences page not retrieved (404 on the guessed URL). Source: https://www.magicbell.com/docs/primitive/user, https://www.magicbell.com/docs/primitive/broadcast
- **Client auth** — HS256 user JWT signed server-side with the project secret key, payload `user_email` and/or `user_external_id` plus `api_key`. Source: https://www.magicbell.com/docs/api/authentication/user
- **Local dev / testing** — Dashboard "Create Broadcast" page to test the Inbox; Events screen for errors; separate projects for staging/production (each extra project beyond plan limit costs $99). Source: https://www.magicbell.com/docs/quick-start
- **Self-hosting** — None found. `magicbell-js` repo is on GitHub (license reported as NOASSERTION) but only the client SDKs.

## Other candidates

### Courier
- Free: 10,000 sends/mo; Business $0.005/send pay-as-you-go; no per-seat or channel fees. Includes in-app channel, CLI, MCP server, SDKs, custom environments listed in the plan table. No self-hosting listed. Source: https://www.courier.com/pricing
- React SDK, trigger API, preferences and read-state details were not verified for this note.

### SuprSend
- Free: 10K notifications/mo, in-app feed with SuprSend branding. Essentials $100/mo (billed annually, branding removed). Business $250/mo billed annually adds user preferences. Self-hosted deployment and BYOC (beta) are Enterprise only. Source: https://www.suprsend.com/pricing
- SDK, API and read-state details were not verified for this note.

## SQLite / self-hosting constraints (cross-cutting)

- Our app uses a single SQLite file via Drizzle (ADR 0006). None of the hosted services touch that DB; they hold their own inbox state and the app only calls the trigger API and mints per-user client credentials (HMAC hash / JWT / signed token).
- Only Novu offers a free self-hosted route in the open-source edition, and it needs MongoDB + Redis + S3, not SQLite.
- All hosted options need a server-held secret (secret API key) and, for production, a server-generated per-user signature for the browser client, which in this app would live in a React Router loader/action.
- Billing units differ: Novu = workflow runs, Knock = messages (per delivery) and guide recipients, MagicBell = deliveries per channel, Courier/SuprSend = sends/notifications.

## Gaps / not verified

- Novu offline/local dev mode; MagicBell preferences API and React Router specifics; Courier and SuprSend client SDKs, read-state and preference docs; rate limits for all vendors.
