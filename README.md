# Browser-Mediated Federated Session Revocation

### A Research-Grade Documentation Package on Logout Enforcement Across Browser State Surfaces

---

## Executive Summary

"Log out everywhere" is one of the most-promised and least-delivered features in modern web security. Users believe that clicking it at an identity provider terminates their sessions everywhere; in practice it usually terminates a server session record at the identity provider and leaves the relying party's client-side state — cookies, stored tokens, service-worker caches, in-memory credentials — fully intact and usable.

This document treats that gap as a real research problem and maps its entire solution space. The central thesis, defended throughout, is that **"session revocation" is not one problem but four**, and that no single layer solves all four:

1. **Local stale-state destruction (P1)** — the relying party's client artifacts persist in *this* browser. *Browser-solvable.*
2. **Credential validity (P2)** — the relying party's server still honors issued credentials. *Server-only, by construction.*
3. **Credential portability (P3)** — the credential was copied off-device before revocation. *Cryptographic binding only.*
4. **Trust and authorization of the revocation signal (P4)** — who may instruct the browser to destroy state for an origin, scoped and authenticated how. *The genuinely open problem.*

The headline conclusions:

- **Browser-only logout is a floor, never a guarantee.** It reliably destroys client state in one browser instance and is powerless against copied credentials and server-honored tokens.
- **Per-account deletion of a relying party's storage is structurally impossible** in a black-box browser, because storage is keyed by origin, not by account. Origin-level destruction is the only robust browser-side default. The honest workaround is the browser profile or container as the per-account boundary.
- **The clean trust anchor is the FedCM grant ledger** (the browser's own record of which sites a user federated into through which provider), with enterprise policy as the closed-world equivalent. A non-browser-owning provider like Facebook or Okta can safely instruct deletion *only* for relying parties it can be shown to have federated — never arbitrary origins.
- **The strongest realizable system is layered**: device-bound credentials (DBSC) for anti-theft, CAEP/Shared Signals for server-side authority, browser-enforced teardown for the local floor, enterprise policy for trust provisioning, and crash-safe journaling for recovery. Each layer is independently useful; none pretends to cover another's job.
- **The most likely thing to fail in practice** is any design that infers federation relationships by watching network flows (privacy-corrosive, lossy) or that skips terminating the provider's own login session (the revocation is silently undone by automatic re-login seconds later).

A note on epistemic status, used consistently below: **[FACT]** = verifiable against current specs, Chromium source, or shipped behavior; **[INFERENCE]** = a reasoned conclusion from facts; **[ASSUMPTION]** = a premise taken as given; **[PROPOSAL]** = a concrete design suggestion of mine; **[SPECULATIVE]** = an idea that does not exist in any engine and may never. Version-sensitive facts are flagged because browser behavior moves quickly.

---

## 1. Plain-English Introduction

### 1.1 What is the problem?

When you sign into a website using "Sign in with Google" (or Apple, Microsoft, Facebook, Okta), two independent things now hold your session open: the website's own record that you are logged in (a cookie, a stored token), and Google's record that you have an active session. These are not linked by any enforcement mechanism. When you later go to Google and click "sign out of all devices," Google ends *its* sessions — but the website you logged into may keep treating you as authenticated indefinitely, because nothing forced it to forget you.

The problem this document studies: **can the browser — the one component that physically holds the website's client-side session state — be made into an enforcement point, so that a trusted signal can destroy that state even if the website never implemented logout properly?**

### 1.2 Why does it matter?

- **Account security.** Stolen session cookies are now a primary attack vector; attackers bypass passwords and multi-factor authentication entirely by replaying a stolen session. A "log out everywhere" that actually reached the client would shrink that window.
- **Device loss and shared terminals.** A user who logs out from home expects the library or hotel computer to be cleared. Today it usually is not.
- **Enterprise deprovisioning.** When an employee is terminated, the organization needs every application session killed promptly, not whenever a token happens to expire.
- **Regulatory.** Data-erasure and timely-access-revocation requirements map directly onto this capability.

### 1.3 Why do users think "log out everywhere" should work?

Because the phrase promises exactly that, and because the mental model is simple: one identity, one logout, everything ends. Users do not distinguish between the identity provider's session and each relying party's independent session, nor between a server forgetting them and their own browser forgetting them. The interface implies a single switch; the reality is dozens of independent switches, most of which the logout button cannot reach.

### 1.4 Why does it often fail in practice?

Four reasons, each a preview of the four core problems:

1. The website's **client state survives** in the browser (the website never wired up logout, or only cleared one cookie). *[P1]*
2. The website's **server still honors** the token it issued, regardless of client state. *[P2]*
3. The credential may have been **copied elsewhere** before logout, where no cleanup can reach it. *[P3]*
4. There is **no trusted, authorized channel** for a non-cooperating relying party's state to be destroyed by anyone but that relying party. *[P4]*

---

## 2. First-Principles Explanation

### 2.1 What is federated login?

**[FACT]** Federated login lets a user authenticate to one service (the **relying party**) using an account held at another service (the **identity provider**), without the relying party ever seeing the user's password. The identity provider vouches for the user's identity by issuing a signed assertion (an ID token in OpenID Connect, an identity assertion in FedCM). The dominant protocols are **OAuth 2.0** (authorization), **OpenID Connect (OIDC)** (authentication layered on OAuth), and **FedCM** (a browser-mediated federation API).

### 2.2 What is a relying party (RP)?

The application the user actually wants to use — `abc.com` in our running example. It "relies" on the identity provider to authenticate the user. After login, the relying party establishes *its own* session, independent of the identity provider's, typically a cookie or a token stored in the browser.

### 2.3 What is an identity provider (IdP)?

The service that authenticates the user and issues assertions — Google, Apple, Microsoft, Okta, Auth0, Facebook, GitHub. It holds its own session with the user (the single sign-on session) and its own record of which relying parties the user authorized.

### 2.4 What does "logout" actually mean in modern web systems?

There is no single definition. "Logout" can mean any combination of:

- deleting the relying party's session cookie in this browser;
- clearing the relying party's tokens from browser storage;
- invalidating the relying party's server-side session record;
- revoking the OAuth tokens at the authorization server;
- ending the identity provider's single-sign-on session;
- tearing down live tabs and workers still holding credentials in memory;
- and, frequently, **none of the above — merely changing what the UI displays** while every credential remains valid. **[FACT]** This last "cosmetic logout" is common and is the reason many "logout" buttons provide no security benefit.

### 2.5 Why is logout not a single action but many?

Because authenticated state is **distributed across independent owners and surfaces**. The browser owns client storage; the relying party owns its server session and decides what its tokens mean; the identity provider owns its single-sign-on session and the federation grant. **[INFERENCE]** A complete logout therefore requires coordinated action across at least three parties and a dozen storage surfaces, and no current standard orchestrates all of them. The naive "delete the cookie" covers one surface owned by one party in one browser.

---

## 3. State Model

This section documents every browser/web surface that can preserve a session. Definitions are given on first use. Each surface is assessed on: what it stores, who can clear it, per-origin clearing, per-account clearing, survival across reload, survival across browser restart, survival across device theft/copying, and collateral damage from clearing.

A foundational concept first:

> **[FACT] StorageKey.** Modern Chromium partitions most storage by a *StorageKey* — composed of the origin, the top-level site, an ancestor-chain bit, and an optional nonce. Storage partitioning by top-level site has been enabled for all Chrome users since Chrome 115. The decisive property for this document: **a StorageKey contains no "account" or "user" dimension.** This single fact makes per-account deletion structurally impossible (Section 8).

> **[FACT] Cookie scoping.** Cookies are scoped to the *registrable domain* (effective top-level domain plus one label — "eTLD+1"), not the exact origin. Clearing cookies for `abc.com` also clears `sub.abc.com`. Only cookies with the `__Host-` prefix are strictly origin-bound.

### 3.1 Per-surface analysis

| Surface | What it stores | Who can clear | Per-origin? | Per-account? | Survives reload? | Survives restart? | Survives theft/copy? | Collateral damage |
|---|---|---|---|---|---|---|---|---|
| **Cookies** (incl. HttpOnly, SameSite) | Session IDs, tokens. HttpOnly hides from JS, not from the browser | RP (its own), browser, extension w/ permission | Registrable-domain, not strict origin | **No** | Yes | Yes (unless session cookie) | **Yes** — a copied cookie works elsewhere | All subdomains + all accounts logged out |
| **Partitioned cookies / CHIPS** | Cross-site cookies scoped to a top-level partition | Same as cookies, per partition key | Per partition | No | Yes | Yes | Yes | Embedded-widget state lost |
| **localStorage** | Key-value strings; often SPA access/refresh tokens | RP (JS), browser | Yes (per StorageKey) | No | Yes | Yes | **Yes** if exported | App preferences/drafts lost |
| **sessionStorage** | Per-tab key-value | RP (JS), browser, tab close | Tab-namespace, not origin-clean | No | Yes (same tab) | No | Yes if exported | Open-tab state lost |
| **IndexedDB** | Structured data, offline tokens, encrypted blobs | RP (JS), browser | Yes (per StorageKey/bucket) | No | Yes | Yes | **Yes** if exported | Offline/PWA data lost |
| **Cache Storage** | Programmatically cached HTTP responses (often authed) | RP (JS/SW), browser | Yes (per StorageKey) | No | Yes | Yes | Yes if copied | Offline app shell re-fetched |
| **Service Worker (SW)** registration + caches | A background script that can intercept all requests and serve cached authed responses offline; runs without a tab | RP (JS), browser (unregister + terminate) | Per scope | No | **Yes — and can resurrect state** | Yes | Registration copies rare; caches yes | PWA/offline broken until reinstall |
| **HTTP cache** | Network-layer cached responses, keyed by network isolation key | Browser | Per network-isolation-key partition | No | Yes | Yes | Yes if profile copied | Re-fetch cost |
| **Live JS / WASM memory** | Tokens loaded into a running tab's heap | **No one can scrub it**; only context destruction | n/a | n/a | **No** (gone on reload) | No | Only if memory-dumped | Unsaved work lost on teardown |
| **In-flight requests** | Requests already carrying an Authorization header | Cancel only | Per origin | n/a | n/a | n/a | n/a | Failed requests |
| **BroadcastChannel / SharedWorker** | Cross-tab messaging / a shared background context holding state | Dies with contexts / terminate worker | Per origin (partitioned) | No | Partially | No | No | Cross-tab coordination resets |
| **bfcache / prerender** | A frozen, fully-live page kept for instant back-navigation / a pre-rendered page | Browser (evict/cancel) | Per origin entries | No | **Restores a live authed page** | No | No | Back-navigation reloads |
| **Extension storage** | Anything an extension chose to store, incl. copied tokens | **Only the extension / its policy** — out of revocation's reach | n/a | No | Yes | Yes | Yes | Extension can re-inject credentials |
| **Profile / container** | A whole isolated identity world (own cookie jar, storage, grants) | Browser/user | n/a | **Yes — the only real per-account boundary** | Yes | Yes | If profile copied | None if used proactively |

### 3.2 Three load-bearing conclusions

1. **[INFERENCE]** Every *persistent* surface is reliably deletable by the browser, but **none is labeled by account**. Deletion is by origin/StorageKey, never by "which user."
2. **[INFERENCE]** *Live memory* (JS/WASM heap, in-flight requests, running workers) cannot be scrubbed; it can only be evicted by destroying the execution context — which is disruptive and reload-based.
3. **[INFERENCE]** *Copyable* surfaces (cookies, localStorage, IndexedDB exports) defeat browser-only revocation entirely once copied, because the copy lives outside the browser's jurisdiction.

---

## 4. Problem Decomposition

The overall issue separates into nine distinct problems. Keeping them separate is the single most important analytical move in this document.

- **P1 — Local stale state.** Client artifacts persist in this browser. *Owner: browser. Solvable.*
- **P2 — Credential validity.** The relying party's server still honors the credential. *Owner: relying party server. Not browser-solvable.*
- **P3 — Credential portability.** The credential was copied off-device before revocation. *Owner: cryptographic binding. Browser holds the key; server does the check.*
- **P4 — Trust and authorization of the signal.** Who may order destruction of an origin's state, authenticated and scoped how. *The open problem.*
  - **P4a — Reauthentication / resurrection.** If the identity provider's session survives, the relying party silently re-logs-in the user, undoing the revocation.
- **P5 — Multi-account precision.** Two users logged into one origin in one browser share one StorageKey; revocation cannot target one.
- **P6 — Offline behavior.** A browser offline at revocation time receives no signal until it reconnects.
- **P7 — Privacy and tracking.** The revocation channel must not become a new way to correlate users or learn which sites they use.
- **P8 — Browser UI / UX consequences.** Forced logout, lost tabs, and "why was I signed out" support load.

**[INFERENCE]** A correct architecture assigns each problem to a layer and refuses to let one layer claim another's territory. Most failed "logout everywhere" promises are a category error: solving P1 (delete the cookie) and claiming P2/P3 (the session is truly dead) were solved too.

---

## 5. Existing Mechanisms and Why They Are Insufficient

For each: what it solves, what it does not, its assumptions, the cooperation it needs, why it fails in realistic deployments, and whether it is a **floor** (minimum local enforcement), a **bridge** (partial/transitional), or a **real solution**.

### 5.1 Clear-Site-Data

**[FACT]** An HTTP response header (`Clear-Site-Data`) by which a site instructs the browser to clear its own `cookies`, `storage`, `cache`, or execution contexts (`executionContexts`), or all (`*`), scoped to the response's origin. **[FACT]** It is honored only on network responses, not service-worker-served responses, to prevent a site clearing arbitrary origins.

- **Solves:** complete client-state destruction for *one origin* when that origin sends the header.
- **Does not solve:** anything when the relying party does not send it; cross-origin; server validity; copied credentials.
- **Assumes / needs:** **relying-party cooperation** (the RP must emit the header).
- **Fails because:** the entire premise of this research is the non-cooperating relying party.
- **Verdict:** the ideal **destruction engine**, but RP-triggered. Reusing its engine under a different, trusted trigger is the heart of the browser-enforcement proposal.

### 5.2 OpenID Connect logout variants (overview)

OIDC defines several logout flows; all assume relying-party implementation.

#### 5.2.1 RP-Initiated Logout
**[FACT]** The relying party redirects the user to the identity provider's `end_session_endpoint` to end the provider session. *Solves:* ending the IdP session when the RP starts it. *Does not solve:* other relying parties' client or server state. *Needs:* RP cooperation. *Verdict:* **bridge.**

#### 5.2.2 Front-Channel Logout
**[FACT]** The identity provider loads each relying party's logout URL in hidden iframes so each RP clears its own session in the browser. **[FACT]** The specification itself warns that the logout iframe "might not be able to access the RP's login state when rendered by the OP in an iframe because the iframe is in a different origin," and that third-party-cookie blocking breaks it. *Solves:* in theory, multi-RP browser logout. *Fails because:* third-party cookie/storage partitioning has largely broken it. *Verdict:* **eroding / [COSMETIC] in modern browsers.**

#### 5.2.3 Back-Channel Logout
**[FACT]** The identity provider sends a signed logout token server-to-server to each relying party's `backchannel_logout_uri`; the relying party invalidates its server session. **[FACT]** The spec notes the RP "must implement an application-specific method of terminating sessions" and that this "can be more complicated than simply clearing cookies and local storage." *Solves:* server-side session invalidation (**P2**) for cooperating relying parties. *Does not solve:* client-side state (the browser never hears about it); copied credentials. *Needs:* RP cooperation. *Verdict:* the right **server-side** mechanism, blind to the client.

### 5.3 FedCM disconnect / Login Status concepts

**[FACT]** FedCM (Federated Credential Management) is a browser API mediating federation without third-party cookies. Relevant pieces: the **Login Status API** (the site informs the browser whether the user is logged in; shipped around Chrome 120); **`IdentityCredential.disconnect()`** (relying-party-initiated disconnection of a federated account, available from around Chrome 122); the **connected-accounts / grant ledger** (the browser's record of (relying party, identity provider, account) triples); and a specification requirement that **the browser must remove connected-account records when a user clears an origin's data.** **[FACT]** FedCM is actively evolving across Chrome releases; treat exact sub-feature availability as version-dependent.

- **Solves:** a browser-held, privacy-reviewed record of which sites federated with which providers — the cleanest *trust anchor* available (Section 8).
- **Does not solve (today):** there is no identity-provider-initiated primitive that destroys a relying party's client state; `disconnect()` is RP-initiated and clears the federation grant, not the RP's session storage.
- **Needs:** the relying party to have used FedCM at all (else no grant exists).
- **Verdict:** **the foundation for the trust/authorization layer (P4)**, not yet an enforcement mechanism.

### 5.4 Polling-based revocation

**[PROPOSAL/INFERENCE]** The browser periodically fetches a signed revocation manifest from the identity provider for federations it recorded. *Solves:* a privacy-friendly (pull, no inbound addressability) transport. *Does not solve:* latency (bounded by interval); offline gaps. *Verdict:* **best transport** for a standardizable design; a **bridge** toward push for low-latency needs.

### 5.5 Push-based revocation (Web Push / SSE / WebSocket)

**[INFERENCE]** Low latency, but a push subscription is a quasi-identifier and a new cross-site channel; Web Push subscriptions belong to the relying party's service worker, reintroducing RP dependence. *Verdict:* **enterprise-only**; privacy-hostile for a consumer standard.

### 5.6 CAEP / Shared Signals Framework / Security Event Tokens

**[FACT]** The OpenID Foundation's **Shared Signals Framework (SSF)** delivers **Security Event Tokens (SETs)** — signed JWTs (IETF RFC 8417) — between a transmitter and a receiver. The **Continuous Access Evaluation Profile (CAEP)** defines event types including **`session-revoked`**. **[FACT]** CAEP 1.0 is published and implemented by major vendors (e.g., Okta, Keycloak); large providers run equivalent continuous-evaluation systems (e.g., Microsoft's Continuous Access Evaluation), where session revocation is enforced in near-real-time with propagation latency that may be on the order of minutes.

- **Solves:** near-real-time **server-side** revocation propagation (**P2**) between cooperating servers.
- **Does not solve:** browser client cleanup — **the receiver is a server, not the browser.** The device physically holding the client state is absent from the architecture.
- **Needs:** transmitter (IdP) and receiver (RP) to implement SSF.
- **Verdict:** the correct **signal semantics and authority** layer; its blind spot — no browser receiver — is precisely the gap this research targets.

### 5.7 Device Bound Session Credentials (DBSC)

**[FACT]** DBSC binds a session to a private key held in device hardware (TPM / secure enclave), so the session cookie is only usable on that device; a copied cookie is worthless elsewhere, and the server gates short-lived cookie refresh on proof-of-possession. **[FACT, version-sensitive]** DBSC reached general availability on Windows in Chrome 146 (April 2026), is enabled by default for Google Workspace users, with macOS secure-enclave support announced as the next target; the specification requires that browsers clear bound sessions and keys when clearing other site data.

- **Solves:** **credential portability (P3)** — the anti-theft layer; and provides a natural server-gated revocation lever (stop refreshing → session dies at short TTL).
- **Does not solve:** local stale state by itself; relying parties that have not adopted it.
- **Needs:** **relying-party adoption.**
- **Verdict:** the **anti-theft layer** of any serious architecture; complementary, not a cleanup mechanism.

### 5.8 Enterprise policy / managed-browser approaches

**[FACT]** Managed Chrome/Edge support administrative policies and managed profiles; administrators can pin trusted configurations and control extensions. **[INFERENCE]** In a managed fleet, the trust-and-authorization problem (P4) is solved by provisioning, full-origin collateral is acceptable, offline devices are reachable on management check-in, and extensions can be restricted.

- **Solves:** trust provisioning and mandated enforcement *within the fleet*.
- **Does not solve:** the consumer web.
- **Verdict:** **the best near-term deployment**; the place where browser-enforced revocation is honest *today*.

### 5.9 Profile separation / containers

**[FACT]** Distinct browser profiles (and Firefox container tabs) have distinct storage and cookie jars. **[INFERENCE]** This is the **only mechanism that gives true per-account isolation** — by prevention, not by selective deletion.

- **Solves:** **multi-account precision (P5)** proactively.
- **Does not solve:** anything retroactively, and depends on user discipline.
- **Verdict:** the honest answer to multi-account; a **floor** for that sub-problem.

### 5.10 Summary table

| Mechanism | Primary problem addressed | Cooperation needed | Floor / Bridge / Real |
|---|---|---|---|
| Clear-Site-Data | P1 destruction engine | RP emits header | Engine (floor when self-triggered) |
| RP-Initiated Logout | IdP session end | RP | Bridge |
| Front-Channel Logout | Multi-RP browser logout | RP | Eroding / cosmetic |
| Back-Channel Logout | P2 server session | RP | Server-side, client-blind |
| FedCM disconnect / Login Status | P4 trust anchor | RP used FedCM | Foundation, not enforcement |
| Polling | Transport | IdP | Bridge (best transport) |
| Push | Transport (low latency) | IdP (+RP for Web Push) | Enterprise-only |
| CAEP / SSF / SET | P2 authority + semantics | IdP + RP servers | Real for P2; no browser receiver |
| DBSC | P3 anti-theft | RP | Real anti-theft layer |
| Enterprise policy | P4 provisioning | none (admin) | Best near-term deployment |
| Profiles / containers | P5 precision | none (user) | Real, by prevention |

---

## 6. Reverse-Engineered Solution Space

Each candidate family below is documented uniformly: idea summary, trust model, deletion target, enforcement point, scope boundary, live-tab/SW handling, multi-account handling, offline behavior, privacy risk, failure modes, best-case use, and residual insufficiency.

### 6.1 Browser-only deletion

- **Summary:** the browser purges an origin's state on some internal trigger.
- **Trust model:** none external — the trigger is internal or extension-driven.
- **Deletion target:** all client surfaces for the origin.
- **Enforcement point:** browser process.
- **Scope boundary:** origin / registrable domain.
- **Live tabs / SW:** must terminate workers and tear down contexts explicitly.
- **Multi-account:** logs out all accounts on the origin.
- **Offline:** trivially local.
- **Privacy:** none (no external channel).
- **Failure modes:** P2, P3 untouched; no authorization story for *who* triggers it.
- **Best case:** the destruction primitive everything else reuses; an extension proof-of-concept.
- **Insufficient because:** it is a mechanism without an authority — it answers "how to delete," never "who may order deletion."

### 6.2 IdP-driven revocation

- **Summary:** the identity provider signals the browser to destroy a relying party's state.
- **Trust model:** signed assertion verified against the provider's published keys (JWKS), *plus a browser-held binding proving the provider federated that relying party.*
- **Deletion target / enforcement / scope:** origin state; browser process; bound (RP, IdP) pairs only.
- **Live tabs / SW / multi-account / offline:** as §6.1, plus suppress automatic re-login.
- **Privacy:** low if the authority derives from data the browser/provider already holds; high if it lets a provider learn or target arbitrary sites.
- **Failure modes:** depends entirely on the binding being trustworthy and scoped (Section 8); P4a if the provider session is not also ended.
- **Best case:** the consumer-web design, anchored on FedCM grants.
- **Insufficient because:** only covers relying parties the browser can bind to the provider.

### 6.3 RP-cooperative server revocation

- **Summary:** the relying party honors a revocation signal and kills its server session (CAEP receiver, introspection, back-channel logout).
- **Trust model:** server-to-server signed events.
- **Deletion target:** the server session / token validity (**P2**).
- **Enforcement point:** relying-party backend.
- **Scope:** the relying party's own sessions, account-precise *server-side*.
- **Live tabs / SW:** indirectly — next request fails once the server rejects the credential.
- **Multi-account:** account-precise on the server (the server knows accounts).
- **Offline:** server enforces on next request.
- **Privacy:** low (no new browser channel).
- **Failure modes:** requires every relying party to implement it; silent on client cleanup.
- **Best case:** the authoritative validity layer.
- **Insufficient because:** it is exactly the cooperation the non-cooperating-RP premise forbids; and it never cleans the client.

### 6.4 Browser-enforced teardown triggered by signed assertions

- **Summary:** §6.1's destruction engine fired by §6.2's verified signal — the synthesis.
- **Trust model:** JWKS-verified SET + browser-held binding (FedCM grant or policy).
- **Everything else:** as §6.2.
- **Best case / why central:** this is the novel contribution — a browser receiver enforcing client destruction on a trusted federated signal without relying-party cooperation. *Insufficient alone* against P2/P3, hence layering.

### 6.5 Polling

As §5.4. Trust via signature on the fetched manifest; privacy-preferred; latency-bounded; offline catch-up on reconnect. The default transport.

### 6.6 Push

As §5.5. Low latency, privacy-risky, enterprise-scoped.

### 6.7 FedCM-anchored trust

- **Summary:** authority to revoke a relying party's state derives from the browser's FedCM grant for (RP, IdP, account).
- **Trust model:** browser-witnessed federation; the strongest privacy posture (the provider already knew these relying parties).
- **Scope:** exactly the federations the browser brokered — bounding blast radius cleanly.
- **Insufficient because:** populated only when the relying party used FedCM; classic OAuth-redirect logins leave no grant.

### 6.8 DBSC + server invalidation

- **Summary:** device-bound short-lived credentials (P3) plus server-gated refusal to refresh (P2).
- **Trust model:** hardware key possession + server policy.
- **Best case:** neutralizes copied credentials and bounds validity to a short TTL.
- **Insufficient because:** requires relying-party adoption; not a client-cleanup mechanism.

### 6.9 CAEP / SSF delivery

As §5.6 — server authority and signal semantics; reused verbatim as the payload for the browser-receiver proposal.

### 6.10 Enterprise-managed enforcement

As §5.8 — the deployment where P4 is solved by provisioning. Best-case near-term path.

### 6.11 Profile / container isolation

As §5.9 — the real answer to P5, by prevention.

### 6.12 OS-level / platform-level assistance

- **Summary:** the operating system or mobile-device-management layer assists — OS account brokers (e.g., Windows Web Account Manager, macOS authentication sessions), keychain-held web credentials, or an MDM remote-wipe of a managed browser profile.
- **Trust model:** platform attestation / MDM channel.
- **Deletion target / scope:** device or OS-account scope; coarse for web origins.
- **Cross-device reach:** **strong** — MDM reaches enrolled devices and queues for offline ones.
- **Privacy:** the OS sees a login graph; contained to the platform.
- **Failure modes:** out of the browser tree; platform-specific; coarse origin precision.
- **Best case:** enterprise fleets and managed mobile; the only family with genuine cross-device reach without server cooperation.
- **Insufficient because:** non-interoperable and coarse; useless on the open consumer web.

### 6.13 Future / browser-native primitives **[SPECULATIVE]**

Three speculative primitives, expanded in Section 12: a standardized **browser session registry + revocation receiver**; **account-keyed authenticated storage buckets** (the only path to per-account precision); and **credential-bound storage encryption** (revocation as key destruction). All require ecosystem shifts; two quietly reintroduce relying-party cooperation.

---

## 7. Bypass and Failure Analysis

Every realistic way the system fails or is bypassed, with the responsible problem class and whether it is prevented, reduced, or only bounded.

- **Copied cookies / tokens (P3).** A credential exported before revocation works elsewhere. *Browser cleanup powerless.* Only DBSC (binding) + server revocation help. **Bounded only.**
- **Tokens already in memory (P1a).** A live tab holding a token in its heap keeps making authenticated calls after storage is wiped. *Mitigation:* destroy the execution context (navigate/reload/kill). **Reduced; sub-second race remains.**
- **Service-worker resurrection (P1a).** A running worker re-creates state or re-registers after a wipe. *Mitigation:* unregister + terminate running versions + re-verify no re-registration before any user gesture. **Reduced.**
- **Stale caches (P1).** Cache Storage or HTTP cache serves an authenticated response post-wipe. *Mitigation:* include both in the purge; flush sockets/HTTP-2/3 sessions. **Prevented if enumerated.**
- **Reauthentication after deletion (P4a).** The page reloads and the still-live provider session silently re-logs-in the user. *Mitigation:* set FedCM login status to logged-out, clear auto-reauthentication, and **end the provider session server-side**. **Reduced; the provider-session kill is mandatory or the revocation is cosmetic.**
- **Account aliasing / same-origin multi-account confusion (P5).** Two accounts share one StorageKey; the browser cannot target one. *Mitigation:* full-origin wipe (logs out both) or profiles/containers. **Structurally unsolved for black-box storage.**
- **Extension reinjection (out of scope).** An extension with host permissions re-injects credentials. *Mitigation:* enterprise extension policy only. **Bounded.**
- **Race conditions during async deletion (P1).** Deletion is asynchronous and multi-process; an in-flight request may complete first. *Mitigation:* order operations (suppress re-auth → terminate workers/cancel in-flight → purge → tear down → re-verify) and block new origin requests until purge confirms. **Reduced to sub-second.**
- **Compromised identity provider (P4).** An attacker with the provider's signing key can forge revocations. *Effect:* equals the provider's existing power to forge identity assertions — no new trust assumption. *Mitigation:* key rotation, rate limits, transparency log. **Bounded to forced-logout DoS of that provider's relying parties.**
- **Spoofed logout messages (P4).** Forged, unsigned signals. *Mitigation:* signature verification + binding gate. **Prevented (modulo verifier bugs — fuzz them).**
- **Replayed logout messages (P4).** An old valid signal resent. *Mitigation:* unique token id cache + issued/expiry timestamps + monotonic per-grant serial; idempotent enforcement makes replays no-ops. **Prevented.**
- **Offline delay (P6).** A disconnected browser never receives the signal. *Mitigation:* journal pending revocations; enforce on reconnect; pair with short server TTLs. **Bounded; gap unbounded without relying-party TTLs.**
- **Third-party cookie restrictions (existing-mechanism failure).** They break front-channel logout iframes. *Effect:* invalidates the legacy multi-RP browser-logout path. **Not mitigable; design around it.**
- **Partitioned storage effects (P1 correctness).** An origin's state exists under multiple StorageKeys (first-party and embedded). *Mitigation:* enumerate *all* StorageKeys whose origin matches, and all cookie partition keys. **Prevented if done correctly; a classic correctness trap.**
- **Untrusted browser state (foundational).** On a shared/public terminal the browser profile itself may be malicious or instrumented. *Mitigation:* none from within; this is why public-terminal scenarios ultimately need server-side revocation. **Bounded.**

---

## 8. Verdicts

Direct answers, each justified.

**Is browser-only logout enough?** **No.** It is a floor. It reliably solves local stale state (P1) in one browser and is powerless against server-honored tokens (P2) and copied credentials (P3). Claiming otherwise is the central error this document exists to refute.

**Is per-account browser deletion actually possible?** **No, for black-box relying-party storage.** Storage is keyed by origin, never by account; values are opaque. The browser cannot distinguish user 1's cookie from user 2's. *Proof sketch:* storage location is a function of (origin, top-level site, ancestor bit, nonce); "account" is not an argument of that function, and the browser does not parse relying-party-internal semantics. The only per-account boundaries are browser-owned objects (a FedCM grant, a DBSC key) and the profile/container.

**Is origin-level deletion the only robust browser-side default?** **Yes.** Because the browser cannot label data by account or confirm a value is "auth," destroying everything for the origin is the only model immune to relying-party storage-schema variation. Selective per-key deletion is fragile and schema-dependent.

**Can DBSC solve the stolen-session problem?** **Largely yes, for adopters.** Binding the session to device hardware makes a copied cookie useless off-device and gives the server a clean revocation lever (stop refreshing). It does not clean local state and does nothing for relying parties that have not adopted it. It is the anti-theft layer, not a logout mechanism.

**Can CAEP solve browser cleanup?** **No.** CAEP solves server-side revocation propagation (P2). Its receiver is a server; the browser is not in the loop. It is necessary for true revocation and insufficient for client cleanup. The proposed fix is to define a *browser receiver* role.

**Can FedCM provide a legitimate trust anchor?** **Yes — the best one available.** The browser's connected-accounts/grant ledger is a privacy-reviewed, browser-witnessed record of which relying parties federated with which providers, and the specification already requires clearing it with site data. It cleanly bounds which relying parties a given provider may act on. Its limit: it exists only where the relying party used FedCM.

**Can a non-browser-owning IdP safely instruct the browser to delete state?** **Yes, but only within a strict boundary.** A provider like Facebook or Okta may trigger deletion *only* for relying parties the browser can independently bind to that provider — via a FedCM grant, an enterprise policy allowlist, or a provider-attested binding list — and only for a signal whose signature verifies against the provider's published keys. The hard rule is **no binding → no action**, which prevents a provider from destroying state for sites it never federated. Inference of bindings by watching network flows is technically possible but privacy-corrosive and lossy, and should be off by default.

**What is the minimum viable standardization path?** A **FedCM extension** adding a provider `revocation_endpoint`, identity-provider-initiated grant-scoped revocation, and mandatory logged-out/auto-reauthentication clearing — carrying a **CAEP `session-revoked` SET** as the verified payload. One working group, one living specification, inherited privacy review.

**What is the best enterprise path?** A **managed-browser feature behind policy**: administrator-pinned trusted providers, mandatory enforcement, the existing destruction stack, paired with provider session termination. Trust provisioning solves P4; full-origin collateral is acceptable in a fleet.

**What is the best consumer path?** The **FedCM-anchored browser-enforcement** design (§6.4/6.7), shipped first behind a flag, scoped strictly to connected accounts, marketed honestly as a client-state floor plus live-tab kill — not "logout everywhere."

**What is the most likely thing to fail in practice?** **Two front-runners.** (1) **Flow-inference binding** — per-tenant provider domains and JavaScript/form-post flows gut its accuracy, and a browser silently building a login graph will not survive privacy review. (2) **Any design that skips ending the provider's own session** — it demos perfectly and is undone by silent re-login seconds later (P4a). Push-based consumer transport is a close third on privacy grounds.

---

## 9. Recommended Architecture

A five-layer design. Each layer is independently valuable and incrementally adoptable; the labeling makes explicit which problem each owns.

- **Layer A — Anti-theft (P3).** Device-bound, short-lived credentials (DBSC; DPoP for token APIs). *Requires relying-party adoption; without it, P3 is uncovered — state this plainly.*
- **Layer B — Server-side validity (P2).** Identity provider emits CAEP `session-revoked` SETs; relying-party receivers kill server sessions where they exist; the **identity provider always ends its own single-sign-on session** (closes P4a). *Identity-provider-cooperative; relying-party-optional.*
- **Layer C — Browser cleanup (P1, the floor, the novel layer).** A browser-process revocation service: verify signal → check binding/policy scope → ordered teardown. *No relying-party cooperation.*
- **Layer D — Enterprise control (P4 provisioning).** Policy-pinned trusted providers, mandatory enforcement, extension restrictions, managed profiles. *Solves trust where it is hardest.*
- **Layer E — Fallback / recovery (P6, P8).** Crash-safe journaling, enforce-on-reconnect, provenance UI, per-provider distrust toggle.

### 9.1 Sequence of operations (Layer C enforcement)

```
On verified, in-scope revocation signal for origin O:

  1. AUTHORITY      Mark FedCM grant for O revoked; set login status = logged-out;
                    clear auto-reauthentication  ............ (prevents silent re-login)
  2. LIVE FIRST     Terminate service workers + shared workers for O;
                    cancel in-flight requests;
                    flush sockets / HTTP-2/3 sessions for O  . (stop resurrection + race)
  3. PURGE          BrowsingDataRemover, registrable-domain filter, mask =
                    cookies (ALL partition keys) + localStorage + sessionStorage
                    + IndexedDB + Cache Storage + SW caches + HTTP cache,
                    enumerating ALL StorageKeys whose origin == O  ... (the destruction)
  4. CONTEXTS       Tear down / navigate all live documents of O;
                    evict bfcache; cancel prerender  ......... (evict in-memory tokens)
  5. VERIFY         Confirm no service worker re-registered;
                    journal completion  ...................... (crash safety)

  Block new network requests for O until step 3 confirms (close the race window).
```

- **Deleted first:** the authority to re-authenticate (grant + auto-reauth), so nothing re-logs-in mid-teardown.
- **Killed / torn down:** workers, in-flight requests, live contexts, bfcache/prerender.
- **Left intact (must never be cleared):** the password manager / saved credentials (revoking a session must not destroy the re-login path), other origins' data, profile-wide settings, history.
- **Journaled:** a "pending revocation for O" record, written before deletion and cleared on confirmed completion.
- **Retried on reconnect:** any journaled revocation not confirmed complete (covers crash and offline).

---

## 10. Chromium / Browser Implementation Sketch

A realistic mapping, not theory. **[FACT]** where a module's role is documented; **[PROPOSAL]** for new code. File paths reflect the current source layout and should be verified before implementation, as the tree moves.

- **Browser process — new `FederatedRevocationService`** **[PROPOSAL]** (a per-profile keyed service): the privileged coordinator. It must live here because only the browser process can verify a signature outside a sandbox and issue privileged cross-origin deletion.
- **FedCM state** `content/browser/webid` **[FACT]**: read connected-accounts/grant records and the provider's key endpoint; revoke grants; clear auto-reauthentication. The authorization substrate.
- **Cookies** `net/cookies` (CookieMonster) via `network::mojom::CookieManager` **[FACT]**: asynchronous, registrable-domain-scoped deletion; **must iterate every cookie partition key**. Runs in the network service.
- **Orchestration** `BrowsingDataRemover` (`content/browser/browsing_data/…`) **[FACT]**: domain-filtered deletion across all data types; the enforcement engine to invoke.
- **Storage fan-out** `StoragePartition::ClearData` / `ClearDataForOrigin` **[FACT]**: StorageKey-scoped deletion of DOM storage, IndexedDB, Cache Storage, service workers; asynchronous; note the documented caveat that the StorageKey matcher does *not* apply to cookies (separate filter).
- **DOM storage** `components/services/storage/dom_storage` **[FACT]**: localStorage per StorageKey; sessionStorage per tab namespace (needs context teardown).
- **IndexedDB** `components/services/storage/indexed_db` **[FACT]**: per-StorageKey/bucket; force-closes open connections.
- **Cache Storage** `content/browser/cache_storage` **[FACT]**: per StorageKey.
- **Service workers** `content/browser/service_worker` **[FACT]**: unregister + `StopAllServiceWorkersForOrigin` + delete for StorageKey; the highest-risk races; re-verify no re-registration.
- **Quota** `components/services/storage/quota` **[FACT]**: single fan-out for quota-managed APIs incl. Storage Buckets — future-proofs surface coverage.
- **StorageKey** `third_party/blink/.../storage_key.h` **[FACT]**: the enumeration key; revocation must query all StorageKeys whose origin matches (first-party and embedded).
- **Network stack** `services/network` **[FACT]**: HTTP cache by network-isolation-key; **flush live sockets/HTTP-2/3 sessions** or a kept-alive authenticated connection survives the wipe.
- **Execution contexts** reuse the Clear-Site-Data `executionContexts` path (`content/browser/.../clear_site_data…`) **[FACT]**: the live-tab teardown primitive; plus bfcache eviction and prerender cancellation.
- **Policy / profile boundaries** `components/policy`, profile services **[FACT/PROPOSAL]**: `FederatedRevocationEnabled`, trusted-provider allowlist, consumer opt-out; the scope ceiling is the profile (never cross-profile).
- **Transport** **[PROPOSAL]**: a per-provider, jittered, ETag-aware poller fetching a signed manifest; an optional Shared-Signals receiver mode behind enterprise policy.
- **Verifier** **[PROPOSAL]**: browser-process-only JWS verification against cached provider keys; issuer/audience/timestamp/token-id/serial checks; per-provider rate limiting; a fuzzed parser.

### 10.1 Implementation path (phased)

1. **Extension proof-of-concept (weeks).** `chrome.browsingData.remove({origins:[…]})` driven by a poll to a provider endpoint. Validates the deletion+transport loop without a fork. **[FACT]** extension `browsingData` clears origin-scoped cookies/storage/cache/service-workers but not extension storage.
2. **Native build behind flag + enterprise policy.** The §9.1 sequence over `BrowsingDataRemover`/`StoragePartition`, FedCM-grant-gated. This is the enterprise product.
3. **FedCM-integrated consumer mode.** Identity-provider-initiated, grant-scoped, with login-status/auto-reauth clearing.
4. **Shared-Signals receiver mode** for managed fleets.
5. **Instrumentation throughout (privacy-safe):** purge latency, race-window hits, service-worker re-registration rate, and the headline metric — **re-login-within-60-seconds rate**.

### 10.2 Test coverage

Browser tests per storage mask (including partitioned cookies and embedded third-party StorageKeys); service-worker-zombie tests; live-tab race measurement; replay/flood/spoof rejection; FedCM auto-reauthentication suppression; crash-recovery (journaled re-enforcement before first load); fuzzing of the token parser.

---

## 11. Standards and Ecosystem Path

- **Browser feature (ship in one engine first):** the destruction engine, the FedCM-grant gating, the verifier, the poller. No standard required to prototype.
- **Web standard (multi-engine):**
  - **Signal semantics** — reuse CAEP `session-revoked` SETs verbatim; propose a **browser-receiver profile** in the Shared Signals working group (a new receiver role, subject-to-origin mapping, browser-appropriate stream/poll config).
  - **Authorization/scope** — a **FedCM extension** in the W3C Federated Identity working group: provider `revocation_endpoint`, identity-provider-initiated grant-scoped revocation, mandatory logged-out + auto-reauthentication clearing. *The minimum viable path.*
  - **Binding for non-FedCM logins** — a small IETF registration for a `.well-known/federated-revocation` location and a SET event type carrying a provider-attested relying-party list.
- **Remain enterprise policy (permanently):** trust provisioning outside FedCM, mandatory enforcement, extension restrictions.
- **Vendor-specific (acceptable, non-interoperable):** a browser vendor's own account-to-browser channel (e.g., a provider that owns the browser) as a low-latency accelerator layered on the standard path.
- **Likely to be adopted:** CAEP/SSF (already are); a FedCM revocation extension (natural fit, active working group, inherited privacy model).
- **Likely to fail review:** spec-level *third-party* Clear-Site-Data triggers (relying-party-autonomy objection — keep it implementation-internal); push-based consumer transport (privacy); any flow-inference binding (privacy); forced cross-origin tab reloads will draw scrutiny as "disruptive and destructive" and need careful UX justification.

---

## 12. Future Research Agenda

### 12.1 Biggest open questions

1. **Where do sessions actually live?** **[INFERENCE — biggest blind spot]** No published corpus measures what fraction of top relying parties are cookie-only versus localStorage-token versus service-worker-cached versus memory-held. This single dataset determines the real-world coverage of any browser floor and would anchor a strong measurement paper.
2. **Can the live-tab race be closed** without killing the renderer process outright?
3. **What revocation latency is acceptable**, and does pull suffice versus push for high-risk cases?
4. **How is the provider single-sign-on session reliably ended** so re-login does not resurrect the relying-party session — and who owns that guarantee?
5. **Cross-engine convergence:** will WebKit and Gecko map a revocation signal to equivalent destruction, given different storage-partitioning models, or diverge?
6. **Governance:** who decides which providers a browser/profile trusts to enforce — the user, the enterprise, or a registry?

### 12.2 Speculative primitives **[SPECULATIVE]**

- **Account-keyed authenticated storage.** Storage written under a bucket tagged to a federated account, so "revoke account A on origin O" deletes only A's bucket. *Would solve the multi-account precision problem (P5).* Catch: the relying party must write into the tagged buckets — quietly reintroducing relying-party cooperation. The Storage Buckets API is a plausible substrate; account tagging and grant-binding are new and not cross-engine.
- **Credential-bound storage encryption.** Origin storage encrypted under a key derived from the session and held in the browser/hardware; revocation **destroys the key**, rendering data unreadable without enumerating and purging it. *Turns deletion into key destruction* — fast, offline-capable, and copy-resistant (ciphertext copies are useless). Catch: deep storage-stack surgery, key-management and performance cost, far from any engine today.
- **Browser-held revocation ledger.** A signed, append-only, monotonic per-grant ledger in the browser recording revocation serials, enabling replay-proof, auditable, offline-coherent enforcement and a transparency story against rogue providers.
- **Stronger per-account storage boundaries.** A first-class "identity" dimension in the StorageKey (beyond profiles/containers) so the platform — not the relying party — enforces per-account isolation. The clean long-term fix for P5; a major platform change with migration and compatibility costs.
- **Logout as an enforceable property.** The unifying research goal: today logout is a relying-party courtesy; the agenda is to make *client-state termination* a property the platform can guarantee on a trusted signal, with the honest caveat that *credential-validity* termination can never be a browser-only guarantee.

### 12.3 Research agenda summary

The measurement study (12.1.1) first, because it sizes every other claim. Then a formalization of revocation correctness as the triple (client-floor, server-validity, portability) with the per-account impossibility proof; quantified race-window and re-login-rate results from a Chromium prototype; and longer-horizon work on account-keyed storage and credential-bound encryption as the only two known routes to per-account precision and deletion-free revocation respectively.

---

## 13. Final Deliverables

**Best current practical architecture.** Enterprise-managed Chromium enforcement: policy-provisioned trusted providers → JWKS-verified CAEP `session-revoked` SET delivered by partitioned polling → grant/policy-scoped full-origin purge with the ordered live-context teardown of §9.1 and crash-safe journaling, **paired with mandatory provider single-sign-on termination**. FedCM-scoped consumer mode follows; DBSC and CAEP adoption are tracked as the layers that lift the system from floor to guarantee.

**Best long-term architecture.** The full five-layer design — DBSC everywhere (anti-theft), CAEP with a standardized browser-receiver role (validity + transport), FedCM-scoped browser enforcement (cleanup + authorization), enterprise policy (provisioning), journaled recovery — augmented, on a multi-year horizon, by the speculative **account-keyed authenticated storage** (to make per-account revocation precise) and **credential-bound storage encryption** (to make revocation a key-destruction operation).

**Biggest unsolved problem.** Trustworthy acquisition of the (identity provider, relying party, account) binding **without any relying-party participation** — the authorization problem (P4). FedCM solves it only where relying parties adopted FedCM; provider-attested manifests solve it at the cost of provider effort and a handed-over login graph; inference solves it at the cost of privacy and accuracy. There is no clean universal answer; every honest design must declare which compromise it accepts. (The runner-up — invalidating a copied credential browser-only — is not *unsolved* but *impossible*, which is a different and more final thing.)

**The one thing most worth prototyping first.** The FedCM-grant-scoped revocation path in Chromium behind a flag (signed-SET poll → verify → ordered purge + teardown + auto-reauthentication suppression), instrumented to measure the two numbers that decide whether the whole idea is real or cosmetic: the **live-tab race window** and the **re-login-within-60-seconds rate**.

**The one thing most worth standardizing first.** The FedCM revocation extension (provider `revocation_endpoint` + grant-scoped enforcement + mandatory logged-out/auto-reauthentication semantics), with CAEP `session-revoked` adopted verbatim as the payload — one working group, one living specification, inherited privacy review.

**The idea most likely to be cosmetic rather than real.** Front-channel logout via hidden iframes (already broken by third-party-cookie/storage partitioning) and, more broadly, any "logout" that changes UI state while leaving credentials valid — together with flow-inference binding, which looks like a clever zero-cooperation trick but collapses under per-tenant provider domains and privacy review.

---

## Appendix A — Glossary

- **Relying Party (RP):** the application a user logs into using a federated identity (e.g., `abc.com`).
- **Identity Provider (IdP):** the service that authenticates the user and issues assertions (e.g., Google, Okta).
- **OAuth 2.0 / OpenID Connect (OIDC):** the dominant authorization / authentication protocols for federation.
- **FedCM (Federated Credential Management):** a browser API mediating federation without third-party cookies; holds a connected-accounts/grant ledger.
- **Connected-accounts / grant ledger:** the browser's record of (relying party, identity provider, account) federations.
- **StorageKey:** Chromium's storage partition key = origin + top-level site + ancestor-chain bit + optional nonce; has **no account dimension**.
- **Registrable domain / eTLD+1:** the cookie-scoping granularity (e.g., `abc.com`), coarser than an origin.
- **Clear-Site-Data:** a response header by which a site clears its own client state.
- **Front-/Back-Channel Logout:** OIDC logout via browser iframes / server-to-server signed tokens, respectively.
- **CAEP / Shared Signals Framework (SSF):** OpenID Foundation specs for delivering signed security events (e.g., `session-revoked`) between servers.
- **Security Event Token (SET):** a signed JWT carrying a security event (IETF RFC 8417).
- **DBSC (Device Bound Session Credentials):** binds a session to a device-held hardware key so copies are useless off-device.
- **DPoP / mTLS:** other sender-constraining mechanisms that bind a token to a key (RFC 9449 / RFC 8705).
- **Service Worker:** a background script that can intercept requests and serve cached responses without a tab.
- **bfcache:** the back/forward cache — a frozen, still-live page kept for instant back-navigation.
- **Live context teardown:** destroying or navigating a tab/worker to evict in-memory state (no heap scrub exists).
- **P1–P8 / P4a:** the problem decomposition of Section 4 (local state; validity; portability; trust/authorization; re-auth resurrection; multi-account; offline; privacy; UX).

## Appendix B — Open Questions (consolidated)

1. Real-world distribution of session storage across top sites (the measurement gap).
2. Closing the live-tab race without killing the renderer.
3. Acceptable revocation latency; pull versus push for high-risk cases.
4. Reliable provider single-sign-on termination and its owner.
5. Cross-engine convergence on destruction semantics.
6. Governance of provider trust (user / enterprise / registry).
7. Whether account-keyed storage or credential-bound encryption can be made deployable without reintroducing full relying-party cooperation.
8. Crash/partial-deletion correctness in practice.
9. Coverage of non-FedCM (classic OAuth-redirect) logins without privacy-corrosive inference.

## Appendix C — Next-Step Experiments

1. **Measurement study.** Instrument a crawler over the top N sites to classify session persistence (cookie-only / localStorage-token / service-worker-cached / memory-held). *Output: the coverage denominator for every claim in this document.*
2. **Extension proof-of-concept.** Origin-scoped purge driven by a polled, signed provider endpoint; measure deletion completeness across surfaces.
3. **Race-window measurement.** Instrument the §9.1 ordered teardown in a Chromium build; quantify the interval in which an in-flight request can still authenticate post-revocation.
4. **Re-login-rate measurement.** With and without provider single-sign-on termination and auto-reauthentication clearing, measure how often a wiped session silently restores within 60 seconds. *This number alone separates a real feature from a cosmetic one.*
5. **Service-worker-zombie test.** Verify no running or re-registered worker serves authenticated responses after enforcement.
6. **Adversarial signal tests.** Forged, replayed, and flooded revocation signals against the verifier; confirm prevention and bounded behavior.

---

*Epistemic-status legend: **[FACT]** verifiable against current specs/source/shipped behavior (version-sensitive items flagged); **[INFERENCE]** reasoned from facts; **[ASSUMPTION]** taken as given; **[PROPOSAL]** the author's design suggestion; **[SPECULATIVE]** does not exist in any engine and may never. Browser behavior is version-dependent and moves quickly; verify version-sensitive facts against current sources before relying on them.*
