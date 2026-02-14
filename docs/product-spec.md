# Product Spec - Encrypted Image Sharing App

## What Is This?

A fast, fun, privacy-first image sharing app where encryption is built into the architecture - not a policy promise. The server never sees your images. Ever.

**One-liner:** "Send images that nobody can see but your recipient - not us, not hackers, not anyone."

> **Technical architecture:** [architecture.md](architecture.md) | **Wireframes:** [app-wireframes.md](app-wireframes.md)

---

## Core Principles

- **Zero-knowledge by design.** The server only stores encrypted blobs. A full database breach yields nothing usable.
- **Provable, not promised.** Open-source client. Users (and auditors) can verify the privacy claims.
- **Fun and fast.** Privacy shouldn't feel like a chore. The UX should feel closer to Snapchat than to a security tool.
- **In and out.** No infinite scroll, no algorithmic feed. Send, view, done.

---

## How It Works

There are two ways to send images. Both use client-side encryption. Neither requires the server to see the image.

### Mode 1: Link Sharing (no account needed)

Anyone can use the app without signing up. This is the entry point and the viral loop.

1. User picks an image, sets expiration and view limit (view once, 24h, 7d)
2. Browser generates a random AES key and encrypts the image on-device
3. Encrypted blob uploads to server
4. User gets a shareable link: `app.com/v/abc123#decryptionKey`
5. The `#fragment` (decryption key) never reaches the server - it stays client-side only
6. Anyone with the link can view the image. No account needed to send or receive.

**Logged-in users can also send via link.** Same flow, but the image is associated with their account so they can manage it (revoke, check view count, delete) from their dashboard.

### Mode 2: Account-to-Account Sharing (both users signed in)

For direct sharing between users who know each other's usernames.

1. Sender looks up recipient's username
2. App fetches recipient's public key from the server
3. Browser generates a random AES key, encrypts the image with it
4. Browser wraps the AES key with the recipient's public key
5. Browser also wraps the AES key with the sender's own public key (dual-wrap — so the sender can re-view what they sent)
6. Uploads: encrypted blob + both wrapped keys
7. Recipient's app unwraps the AES key using their private key and decrypts on-device

The sender never shares a link or fragment. The recipient's key pair handles access.

#### Sender's Copy

In account-to-account mode, the sender's client wraps the AES key with their own public key in addition to the recipient's (step 5 above). This means the sender can decrypt and re-view their own sent images from the dashboard.

Without this, the "Sent" tab would just be metadata with lock icons — the sender could never revisit what they sent. The extra wrapped key is ~256 bytes per image and leaks no additional information since the server already knows the sender in account mode.

### Encrypted Payload

The client doesn't encrypt raw image bytes. It encrypts a structured payload that bundles the image with metadata the server should never see — including the sender's name, the image type, and a reserved caption field. This is serialized and encrypted as one blob. The server only ever sees the outer ciphertext.

For the payload format and encryption details, see [architecture.md](architecture.md#22-encrypted-payload-format).

### Viewing

- **Via link:** Open the link → browser reads the decryption key from the `#fragment` → fetches encrypted blob → decrypts → displays. No account needed.
- **Via account:** Open inbox → app fetches blob + wrapped key → unwraps with private key → decrypts → displays.

### Notifications

Account-to-account images need a way to reach the recipient. The options and their trade-offs:

| Option             | Pros                          | Cons                                          |
| ------------------ | ----------------------------- | --------------------------------------------- |
| Pull-based inbox   | Zero metadata leakage, simple | Not real-time, user must open app             |
| Web Push (opt-in)  | Native-feeling, real-time     | Unreliable on iOS, permission fatigue         |
| Email notification | Reliable delivery             | Adds metadata trail, undermines privacy story |

**Decision:** Pull-based inbox is the primary mechanism — users open the app and see new items. Web Push is offered as an opt-in with generic content only ("You have a new image on Hush" — no sender name, no preview). No email notifications — the metadata trail contradicts the privacy promise.

PWA badge API (where supported) shows unread count on the app icon without a push notification.

### Flow Diagrams

For detailed sequence diagrams of all flows (link sharing, account-to-account, signup/login, password change, account recovery), see [architecture.md](architecture.md#9-appendix-sequence-diagrams).

---

## User-Controlled Lifecycle

The sender chooses how long content lives. This isn't Snapchat deciding for you. These controls are available in both link and account-to-account sharing.

| Option             | Behavior                                                                    |
| ------------------ | --------------------------------------------------------------------------- |
| **View once**      | Image key discarded after first view; server deletes blob                   |
| **Time-limited**   | Expires after 24h or 7d                                                     |
| **Persistent**     | Stays until sender deletes it (account-to-account only)                     |
| **Revoke anytime** | Sender can kill access instantly regardless of the above (requires account) |

When the encrypted blob is deleted, the content is mathematically unrecoverable.

Non-logged-in users can set expiration and view limits at send time. Logged-in users get full lifecycle control including persistent storage and revocation.

### View-Once: Sender Behavior

If a sender creates a view-once link and then clicks it themselves, they'd consume the single view — leaving the intended recipient with "This image has expired."

- **Logged-in sender:** The sender's own click does NOT count against the view limit. The server knows the sender's account and exempts them. A banner shows: "This is your image. This view doesn't count."
- **Anonymous sender:** No way to identify the sender. The Link Created screen warns prominently: "View-once means the first person to open this link sees the image — including you."
- **Account-to-account:** Not applicable — the sender views their own copy via the dual-wrapped key in their Sent tab, not via a link.

---

## Accounts, Auth & Key Management

Accounts are optional. The app is fully usable without one. Accounts unlock direct sharing, persistent storage, revocation, and a personal dashboard.

### Signup

1. User signs up with email + password
2. Email verification: server sends a verification code. User must verify before the account is fully activated. Until verified, the user can still use anonymous link sharing but cannot send account-to-account. This is the only email Hush ever sends.
3. Browser generates a public/private key pair
4. Public key → stored on server in plaintext (it's meant to be shared)
5. Private key → encrypted with a key derived from the user's password → stored on server as an encrypted blob
6. Server never sees the raw private key
7. User is shown a recovery phrase - the only way to recover if they lose their password
8. No password reset. No backdoor. That's the point.

For the detailed cryptographic signup flow (key derivation, recovery phrase generation), see [architecture.md](architecture.md#61-signup-flow).

### Login (every session, every device)

1. User enters email + password
2. Server authenticates normally (password hash check), issues a session token
3. Server sends back the encrypted private key blob
4. Browser re-derives the key from the password and decrypts the private key into memory
5. Private key is now available in browser memory for the session

**The password does double duty** - it authenticates the user _and_ unlocks their private key. The user experiences this as a single step: type password, you're in.

For the detailed cryptographic login flow, see [architecture.md](architecture.md#62-login-flow). For password change and account recovery flows, see [architecture.md](architecture.md#63-password-change-flow) and [architecture.md](architecture.md#64-recovery-phrase-mechanism).

### Key Lifecycle

- One key pair per account, persistent forever
- Same key pair works across every device - logging in on a new device just re-derives and decrypts
- Recovery phrase can regenerate the private key if the password is lost
- No key rotation (conscious trade-off — rotation would require re-encrypting all wrapped keys for existing images)

### Session Management

- **Active sessions list:** Users can see all active sessions (device type, last active, approximate location) in Settings
- **Remote logout:** Users can terminate any session from the list. Termination invalidates the session token and clears the decrypted private key from that client's memory on its next server communication.
- **Log out all devices:** One-tap option to invalidate all sessions except the current one
- **Password change:** Automatically invalidates all other sessions (forces re-login with new password)

### Username Changes

The Profile screen has "Edit username." The implications:

- **Existing shares:** Unaffected. Account-to-account shares are keyed on internal account ID, not username.
- **Sender attribution:** The sender name embedded in previously-sent encrypted payloads is baked into the ciphertext. Old images will show the old username. New images show the new one.
- **Old username:** Reserved for 30 days after change, then released. Another user can claim it after the reservation period.
- **QR codes:** Encode the username. Old QR codes go stale — user should regenerate/re-share.
- **Rate limit:** One username change per 30 days.

### Account Deletion

Settings includes a "Delete account" button. The consequences:

- **Sent images (link-based):** Active links continue to work until expiry — they don't depend on the sender's account. The sender loses dashboard management (revoke, delete).
- **Sent images (account-to-account):** Encrypted blobs remain accessible to recipients who already have the wrapped key. The sender's wrapped key copy is deleted.
- **Received images:** The user's private key is destroyed. Any received images not yet decrypted become permanently inaccessible. Client-side cached content is cleared.
- **Username:** Reserved for 30 days to prevent impersonation. After the grace period, released for others to claim.
- **Server data:** Public key, encrypted private key blob, and recovery blob are deleted. Blob metadata for active shares is retained until blob expiry, then purged.

**Confirmation UX:** Multi-step — tap "Delete account" → warning screen explaining consequences → type username to confirm → account deleted → redirect to landing page.

---

## No Discovery

Users cannot be discovered, searched, or suggested by the platform. The server does not know who knows who.

Users connect out-of-band:

- Share a username in person or via text
- Scan a QR code
- Send a link (the primary viral loop)

**Every encrypted link shared with a non-user is a product demo.** The view page is the growth engine.

---

## What the Server Knows vs. Doesn't Know

| Server knows                                                   | Server does NOT know           |
| -------------------------------------------------------------- | ------------------------------ |
| That a user account exists                                     | Image content                  |
| That an encrypted blob was uploaded                            | Decryption keys                |
| Blob sizes and timestamps                                      | User's private key             |
| Public keys                                                    | Who the recipient of a link is |
| That User A shared something with User B (account shares only) | What was shared                |

---

## Security Model

**What we protect against:** server breaches, insider threats, and subpoenas. The server never holds decryption keys. A full database dump yields only encrypted noise.

**What we can't protect against:** the recipient. Once an image is decrypted on someone's screen, they can screenshot, screen record, or save it. This is true of every messaging app. We're honest about it.

### Known Attack Vectors

| Attack                                                              | Content exposed?         | Mitigation                                                                         |
| ------------------------------------------------------------------- | ------------------------ | ---------------------------------------------------------------------------------- |
| Server / infra breach                                               | No                       | ZK architecture - server only has ciphertext                                       |
| Link interception (shared device, clipboard, compromised messenger) | Yes, that image          | View-once, short expiry. Link sharing is only as secure as the channel carrying it |
| Screenshot / screen recording                                       | Yes, viewed image        | No technical prevention in PWA. Be transparent with users                          |
| Compromised client JS (CDN MITM, malicious extension)               | Potentially all          | SRI hashes, reproducible builds, published release hashes, eventual native shells  |
| Weak password + server breach                                       | All account images       | Enforce strong passwords, aggressive Argon2 parameters (1-2s derivation)           |
| Recovery phrase theft                                               | All account images       | Clear UX guidance to store securely                                                |
| Metadata analysis                                                   | Content no, patterns yes | Accepted Tier 1 trade-off. API logs minimized, optional blob size padding later    |
| Expired but not yet purged blobs                                    | No (API enforces expiry) | Frequent R2 cleanup jobs. Blob without key is useless regardless                   |

### The Promise

"We protect your images from us and from breaches. We can't protect them from the person you chose to share them with."

---

## Trust & Safety

E2E encryption means the server cannot inspect content. This is the product's core promise — and it creates a tension with content moderation. Here's how we handle it.

### Content Moderation Position

Hush cannot and does not scan encrypted content. This is by design and is the foundation of the privacy guarantee. Client-side hash-based scanning (e.g., PhotoDNA before encryption) is architecturally possible but is not included — it would require trusting the client to self-report, which defeats the purpose and creates a precedent for expanding scanning.

### User Reporting

A report button is available on the view image screen (both link-view and account-view).

- **What a report contains:** blob ID, reporter's account (if any), timestamp, report reason category (illegal, harassment, spam, other)
- **What a report does NOT contain:** the decrypted image — the server cannot see it
- Report flow: user taps report → selects reason → confirmation
- Backend: flag the blob metadata. If a report threshold is reached, restrict access (blob fetch returns `451`) pending review
- "Review" is limited to metadata (blob size, timestamps, sender account if known) since content is encrypted

### Abuse Response

- Repeated reports against a sender account trigger an escalation ladder: warning → temporary send restriction → account suspension
- Suspended accounts: public key remains (existing recipients can still decrypt), but sending capability is revoked
- Law enforcement requests: Hush can provide metadata (account existence, timestamps, IP logs if retained) but cannot provide image content

### Rate Limiting

Free and pro tiers have rate limits on sends to prevent spam at scale. See [architecture.md](architecture.md#8-rate-limiting--abuse-prevention) for specific limits.

### App Store & Legal

- Apple and Google require the ability to respond to reports — the reporting mechanism satisfies this
- Age gate: 13+ (or applicable local minimum)
- Transparency report commitment: aggregate numbers only (total reports, actions taken), published periodically

---

## Blocking & Consent Controls

### The Problem

Anyone who knows a username can send unsolicited encrypted images. This is the AirDrop problem — and it must be addressed before launch.

### Block User

- Any logged-in user can block another username
- Blocked sender's messages are silently dropped server-side — the server refuses to store the blob for that sender→recipient pair
- Block list stored server-side (the server already knows "User A shared with User B" in account mode, so this leaks no new information)
- Unblock available from Settings
- Block action offers "Block and report" as a combined option, feeding into the Trust & Safety reporting pipeline

### Accept/Reject Flow (Deferred)

An optional "message request" mode where first-time senders require recipient approval before the blob is delivered (similar to Instagram DM requests). This adds friction to the core sharing loop and is deferred — blocking is sufficient for handling unwanted content.

---

## Brand

- **Tone:** Fun, modern, bold, playful
- **Feel:** Snappy interactions, minimal screens, bold colors, quick in-and-out
- **Design reference:** BeReal / early Instagram energy - not dark mode hacker aesthetic
- **Trust surface:** Subtle lock/shield icon on images. Tap for details. Never preachy.

---

## Accessibility

Hush targets WCAG 2.1 AA compliance across all screens. Privacy tools should be usable by everyone.

- **Color contrast:** All text meets 4.5:1 contrast ratio minimum. Interactive elements meet 3:1.
- **Screen reader support:** All interactive elements have descriptive ARIA labels. Lock/shield icons have alt text. Encrypted/decrypted state is communicated non-visually.
- **Keyboard navigation:** Full keyboard navigability on desktop. Logical tab order. Visible focus indicators.
- **Touch targets:** Minimum 44x44px tap targets on mobile.
- **Motion:** Respect `prefers-reduced-motion`. Encryption progress animations can be replaced with static indicators.
- **Font scaling:** UI accommodates up to 200% text zoom without horizontal scrolling.
- **Error messages:** Associated with form fields programmatically, not just by color.

---

## Competitive Positioning

- **vs. Snapchat** - They promise privacy. We prove it. They see everything server-side.
- **vs. Signal** - They do encrypted text. We do encrypted visual sharing with personality.
- **vs. Telegram** - They default to plaintext. We're E2E always, by architecture.

---

## Revenue

**Freemium.** Encryption is never gated - free users get the same ZK guarantees.

|                   | Free        | Pro          |
| ----------------- | ----------- | ------------ |
| Encrypted sharing | ✓           | ✓            |
| Storage           | Limited     | Expanded     |
| Max file size     | 5 MB        | 50 MB        |
| Expiry options    | 24h, 7d     | All + custom |
| Account shares    | Limited/day | Unlimited    |
| Albums/groups     | -           | ✓            |

**Contextual (non-targeted) ads.** Based on screen placement, time, region - never on content. _"Our ads are private too."_

---

## Image Limits & Supported Types

**Size limits** are constrained by client-side browser memory during encryption, not by storage (R2 supports up to 5TB). These limits are well within safe territory for all devices.

|               | Free | Pro   |
| ------------- | ---- | ----- |
| Max file size | 5 MB | 50 MB |

**Supported types:** JPEG, PNG, WebP, GIF (including animated)

**Excluded:** SVG (XSS vector - can contain embedded scripts), AVIF (inconsistent browser support), BMP, TIFF. Revisit as browser support matures.

The server is type-agnostic - it only stores encrypted bytes. Type validation and rendering happen entirely on the client.

---

## Tech Stack (High-Level)

| Layer        | Choice                                                                 |
| ------------ | ---------------------------------------------------------------------- |
| Frontend     | TanStack Start (React), PWA, shared responsive UI for mobile + desktop |
| API          | Hono on Cloudflare Workers                                             |
| Blob storage | Cloudflare R2 (encrypted blobs)                                        |
| Metadata DB  | Cloudflare D1                                                          |
| Encryption   | Web Crypto API                                                         |
| Validation   | Zod (shared client/server schemas)                                     |
| Language     | TypeScript everywhere                                                  |
| Client       | Open source                                                            |

For crypto primitives, API design, database schema, and implementation details, see [architecture.md](architecture.md).

---

## Pages (Logged-In)

Seven screens total. Bottom nav on all screens: Dashboard, Profile, Settings.

### Dashboard (Sent)

```
┌─────────────────────────────────┐
│  DASHBOARD                      │
│                                 │
│  [+ Send Image]                 │
│                                 │
│  Sent                  Received │
│  ─────────────────────────────  │
│                                 │
│  ┌─────┐ To: @jane              │
│  │ 🔒  │ Expires: 23h left       │
│  │     │ Views: 3               │
│  └─────┘ [Revoke]               │
│                                 │
│  ┌─────┐ To: Link               │
│  │ 🔒  │ View once - used        │
│  │     │ Views: 1/1             │
│  └─────┘ [Delete]               │
│                                 │
│  ┌─────┐ To: @mark              │
│  │ 🔒  │ Persistent              │
│  │     │ Views: 12              │
│  └─────┘ [Revoke] [Delete]      │
│                                 │
│             [─] [@] [⚙]         │
└─────────────────────────────────┘
```

No thumbnails - server can't generate previews since it only has ciphertext. Lock icon placeholder for every image.

### Dashboard (Received)

```
┌─────────────────────────────────┐
│  DASHBOARD                      │
│                                 │
│  [+ Send Image]                 │
│                                 │
│  Sent                  Received │
│  ─────────────────────────────  │
│                                 │
│  ┌─────┐ From: @jane            │
│  │ 🔒  │ Tap to decrypt          │
│  │     │ Expires: 6h left       │
│  └─────┘                        │
│                                 │
│  ┌─────┐ From: @alex            │
│  │ 🔒  │ View once               │
│  │     │ Tap to view            │
│  └─────┘                        │
│                                 │
│  ┌─────┐ From: @mark            │
│  │  ✓  │ Viewed 2m ago          │
│  │     │ Persistent             │
│  └─────┘                        │
│                                 │
│             [─] [@] [⚙]         │
└─────────────────────────────────┘
```

Sender name in the inbox list comes from server metadata (the server knows sender→recipient in account mode). The sender name shown on the decrypted image view comes from the encrypted payload. See [architecture.md](architecture.md#54-server-metadata-per-sharing-mode) for the full breakdown of what the server knows per mode.

### Send Image

```
┌─────────────────────────────────┐
│  SEND IMAGE                     │
│  <- Back                        │
│                                 │
│  ┌─────────────────────────┐    │
│  │                         │    │
│  │    Tap to pick image    │    │
│  │    or drag & drop       │    │
│  │                         │    │
│  └─────────────────────────┘    │
│                                 │
│  Send to:                       │
│  ( ) Link - anyone with link    │
│  ( ) User - @username           │
│                                 │
│  Expires:                       │
│  [View once] [24h] [7d] [None]  │
│                                 │
│  Max views: [ __ ] (optional)   │
│                                 │
│  [ Send securely ]              │
│                                 │
│             [─] [@] [⚙]         │
└─────────────────────────────────┘
```

### Link Created

```
┌─────────────────────────────────┐
│  LINK CREATED                   │
│  <- Back                        │
│                                 │
│                                 │
│           ✓ Encrypted           │
│                                 │
│  ┌─────────────────────────┐    │
│  │ app.com/v/abc123#k3y... │    │
│  └─────────────────────────┘    │
│                                 │
│  [ Copy link ]  [ Share... ]    │
│                                 │
│  Expires: 24h                   │
│  Max views: 1                   │
│                                 │
│  This link contains the         │
│  decryption key. Anyone with    │
│  it can view the image.         │
│                                 │
│             [─] [@] [⚙]         │
└─────────────────────────────────┘
```

### View Image

```
┌─────────────────────────────────┐
│  VIEW IMAGE                     │
│  <- Back                        │
│                                 │
│  Decrypting...                  │
│  ████████████░░░░ 75%           │
│                                 │
│  ┌─────────────────────────┐    │
│  │                         │    │
│  │                         │    │
│  │      [ decrypted        │    │
│  │        image ]          │    │
│  │                         │    │
│  │                         │    │
│  └─────────────────────────┘    │
│                                 │
│  From: @jane                    │
│  Expires: 5h left               │
│  🔒 Encrypted end-to-end         │
│                                 │
│             [─] [@] [⚙]         │
└─────────────────────────────────┘
```

"From" sender name extracted from decrypted payload.

### View Image (via link, no account)

This is the most important growth surface — every shared link is a product demo.

```
┌─────────────────────────────────┐
│  [H] hush                       │
│                                 │
│  Decrypting...                  │
│  ████████████░░░░               │
│                                 │
│  ┌─────────────────────────┐    │
│  │                         │    │
│  │      [ decrypted        │    │
│  │        image ]          │    │
│  │                         │    │
│  └─────────────────────────┘    │
│                                 │
│  From: @jane                    │
│  View once                      │
│  🔒 Encrypted with Hush         │
│                                 │
│  ┌─────────────────────────┐    │
│  │  Send your own           │    │
│  │  → Create account        │    │
│  └─────────────────────────┘    │
│                                 │
└─────────────────────────────────┘
```

Key elements: Hush branding, decrypted image, sender attribution from decrypted payload, trust badge, and the **CTA: "Send your own → Create account"** — the conversion moment.

### Profile

```
┌─────────────────────────────────┐
│  PROFILE                        │
│  <- Back                        │
│                                 │
│       ┌───┐                     │
│       │ 😀│  @username          │
│       └───┘                     │
│                                 │
│  ┌─────────────────────────┐    │
│  │  QR Code                │    │
│  │  ████████████           │    │
│  │  █          █           │    │
│  │  ████████████           │    │
│  │                         │    │
│  │  Scan to add me         │    │
│  └─────────────────────────┘    │
│                                 │
│  [ Edit username ]              │
│  [ Show recovery phrase ]       │
│                                 │
│             [─] [@] [⚙]         │
└─────────────────────────────────┘
```

QR code is the primary in-person sharing mechanism since there's no discovery.

### Settings

```
┌─────────────────────────────────┐
│  SETTINGS                       │
│  <- Back                        │
│                                 │
│  Account                        │
│  ─────────────────────────────  │
│  Change password                │
│  Show recovery phrase           │
│  Active sessions         ->     │
│  Blocked users           ->     │
│  Export public key              │
│                                 │
│  Defaults                       │
│  ─────────────────────────────  │
│  Default expiry      [ 24h  v]  │
│  Default max views   [ None v]  │
│                                 │
│  About                          │
│  ─────────────────────────────  │
│  What we can't see    ->        │
│  Source code (GitHub)  ->       │
│  Security whitepaper   ->       │
│                                 │
│  ─────────────────────────────  │
│  [ Delete account ]             │
│  [ Log out ]                    │
│                                 │
│             [─] [@] [⚙]         │
└─────────────────────────────────┘
```

> For detailed mobile and desktop wireframes including all screen variants, see [app-wireframes.md](app-wireframes.md).

---

## Error & Empty States

Every state the user might encounter needs a designed response. These affect trust and perception as much as the happy path.

### Expired Link

Shown when a view-link is opened after expiry or view limit is exhausted.

- Message: "This image has expired."
- Subtext: "The sender set this image to expire after [duration]."
- No indication of what the image was or who sent it
- CTA: "Send your own → Create account" (growth opportunity even on error)

### Decryption Failure

Shown when the `#fragment` is missing, corrupted, or the blob can't be decrypted.

- Message: "Couldn't decrypt this image."
- Guidance: "Make sure you copied the full link, including everything after the # symbol."
- This is the most common user error — partial URL copy from messaging apps that strip fragments

### User Not Found

Shown when sending to a username that doesn't exist.

- Inline validation on the username field: "User not found"
- Do NOT reveal whether the username was ever registered (timing-safe lookup to prevent enumeration)

### Empty Dashboard (New User)

Shown when a newly signed-up user lands on the dashboard with no sent/received items. This is the onboarding moment.

- Friendly illustration + message: "Nothing here yet. Send your first encrypted image."
- Prominent "+ Send Image" CTA
- Brief value reinforcement: "Images you send and receive will appear here."

### Upload Failure

Shown when encryption succeeds but the upload fails (network error, server error).

- Message: "Upload failed. Your image was encrypted but couldn't be sent."
- "Try again" button — the encrypted blob can be re-uploaded without re-encrypting
- Reassurance: "Your image never left your device unencrypted."

---

## Scope

### Included

- Sign up / log in with client-side key generation
- Email verification on signup
- Recovery phrase on signup
- Send encrypted image via shareable link
- Send encrypted image to a username (with dual-wrap for sender preview)
- View page: beautiful, fast, the "wow moment" — with conversion CTA for non-users
- Lifecycle control: view-once, 24h, 7d, persistent, revoke
- Notifications: pull-based inbox + optional Web Push
- Block user
- Active sessions / remote logout
- Error and empty states for all flows
- Trust & safety: reporting, rate limiting, abuse response
- Accessibility (WCAG 2.1 AA)
- PWA installable on mobile and desktop
- Transparency page ("what we can and can't see")
- Open-source client

### Deferred

Conscious trade-offs, not a roadmap — each item has a reason it's excluded.

- Multi-image / albums
- Group sharing
- Reactions, replies, read receipts
- Pro tier / payments / ads
- Screenshot detection
- Native app shells
- Contact discovery or syncing
- Backup / export
- Accept/reject flow for first-time senders (adds too much friction to the core loop)
- Client-side content scanning (incompatible with ZK architecture)
- Email notifications (metadata trail contradicts privacy promise)
- High-contrast theme / RTL support
- Key rotation (would require re-encrypting all wrapped keys for existing images)
