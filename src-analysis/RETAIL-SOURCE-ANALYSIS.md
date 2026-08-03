# White-Label Player Site — Source Analysis

**Target:** `https://white-label-2.gammaplus.io`
**Method:** Static reverse-engineering of the SSR HTML + downloaded `_next/static` chunks, plus live browser inspection (network/console/DOM) via an authenticated demo session.
**Analysis directory:** `/Users/artem/Projects/casino-management/src-analysis/retail/`

---

## 1. Download Summary

| Asset | Count | Size |
|---|---|---|
| Homepage SSR HTML (`index.html`) | 1 | 1.1 MB |
| Route SSR HTML snapshots (`pages/`) | 13 pages fetched, 7 valid (see §2) | 6.2 MB |
| JS chunks (`chunks/`) | 41 files | 1.3 MB |
| CSS bundles (`css/`) | 8 files | 404 KB |
| Build manifest (`manifest/_buildManifest.js`, `_ssgManifest.js`) | 2 | 8 KB |
| i18n locale objects (extracted from RSC flight payload, `i18n/en-home-full.json`) | 8 locales | 156 KB |
| **Grand total downloaded** | | **~9.1 MB** |

`_buildManifest.js` / `_ssgManifest.js` are effectively empty — this is a pure App Router build (no `pages/` router, no static-generation manifest of note; the classic `_buildManifest` is a legacy stub `{"/_app":..., "/_error":...}`). Build ID: `YCZ3_mcYG6kfFUa0UJjd0`.

Note on locale JSON: the `require.context` map in `layout-180563d97be75b95.js` (webpack module `90492`) references per-locale `home.json` chunks (e.g. chunk id `9105` for `en`) that are *not* independently fetchable at the URLs implied by the client-side webpack chunk-id→hash table (`webpack-*.js`) — those ids collide with a different chunk map and 404'd. The full English (and 7 other locales') translation tree was instead recovered intact from the React Server Components flight data (`self.__next_f.push(...)`) embedded in the SSR HTML — this is actually a *better* source since it's the fully-resolved, hydration-ready object.

---

## 2. Route Map

The app is Next.js **App Router** with a `[locale]` dynamic segment and a **`layout2`** named route group — confirmed directly by a response header on every page:

```
x-middleware-rewrite: /en/layout2/casino
```

i.e. Next middleware rewrites the public URL `/en/casino` internally to `/en/layout2/casino`. The physical chunk paths confirm the same structure:

```
app/[locale]/layout2/layout-180563d97be75b95.js
app/[locale]/layout2/page-4a87a6f260719bfb.js            (home)
app/[locale]/layout2/casino/page-a5e36caee330b69f.js
app/[locale]/layout2/login/page-b5a981c03d88a153.js
app/[locale]/layout2/register/page-ff9d38fc26603f87.js
app/[locale]/layout2/support/page-79acdb2fc913abb4.js
app/[locale]/layout2/play-casino/[...slug]/page-83b6a91a0ee055ef.js
app/[locale]/layout2/error-a1ef4e9dc56b5e80.js
```

**"layout2" is a theme/skin selector, not a page-content route.** The name is confirmed *not* to be a marketing/feature slug — it's directly visible in the middleware rewrite and in the module folder structure. Combined with the bundled i18n data (§5), which contains two literal boilerplate keys `layout1`/`layout2` (translated as `"Layout1"/"Layout2"`, `"Disposition1"/"Disposition2"` in FR, etc. — leftover strings from an i18next-Next.js starter template, not real UI copy), this strongly implies GammaStack ships **multiple selectable front-end layouts/skins** (at minimum `layout1` and `layout2`) as part of its white-label product, and this operator instance is configured to run `layout2`. This is the mechanism by which different white-label brands running on the same GammaStack platform get visually distinct front ends while sharing the same API/backend and translation infrastructure.

### Probed routes (HTTP status against the live instance)

| Route | Status | Notes |
|---|---|---|
| `/en` → `/` | 200 | Locale redirect strips default locale `en` from the URL |
| `/en/casino` → `/casino` | 200 | Game lobby |
| `/en/login` → `/login` | 200 | |
| `/en/register` → `/register` | 200 | |
| `/en/support` → `/support` | 200 | |
| `/en/bonus` → `/bonus` | 200 | |
| `/en/tournaments` → `/tournaments` | 200 | |
| `/en/play-casino/test-game` | 200 (shell only, no valid game) | Real pattern is `/play-casino/[...slug]`, see §4 |
| `/en/account/*` (kyc, profile-details, wallet/deposit, wallet/withdraw, wallet/transactions, wallet/casino-transactions, wallet/my-bonuses) | 307 → `/en/login` | Auth-gated, redirect to login when unauthenticated |
| `/en/sports` | 307 → `/en/login` | Auth-gated |
| `/en/all`, `/en/promotions`, `/en/vip`, `/en/responsible-gambling`, `/en/live-casino`, `/en/providers` | 307 → locale-stripped path → **404** | Routes exist in the compiled route tree / are linked from other layout templates, but are **not enabled/configured on this specific operator instance** — good evidence that GammaStack feature-flags whole pages per white-label deployment rather than only toggling UI elements |

### Routes discovered by crawling (from chunk names + live navigation)

```
/                                          (home)
/casino                                    (lobby, ?provider=<id> filter)
/login
/register
/support
/bonus
/en/bonus/all?bonusType=all                (bonus listing with query filter — reached via live nav)
/tournaments
/play-casino/[gameId]/[providerId]/[gameName]/[demoAvailable]   (game player — see §4)
/account/profile                            (authenticated landing, seen live)
/account/my-account/kyc
/account/my-account/profile-details
/account/wallet/casino-transactions
/account/wallet/deposit
/account/wallet/my-bonuses
/account/wallet/transactions
/account/wallet/withdraw
```

---

## 3. Tech Stack

| Layer | Finding |
|---|---|
| Framework | Next.js App Router, `x-powered-by: Next.js`, RSC streaming (`self.__next_f.push`), **Server Actions** heavily used (see §4/§6). No version string exposed anywhere in markup or bundles (modern Next versions stripped the `generator` meta tag); feature set (stable Server Actions, RSC, middleware rewrites) implies **Next.js 13.4+/14.x**. |
| UI | React (found `ReactCurrentOwner`/`SECRET_INTERNALS` internals in `fd9d1056-e957847b4709a318.js`, the framework runtime chunk); no explicit version string bundled. |
| State management | **Redux Toolkit** + `react-redux` (`redux-toolkit.js.org`/`redux.js.org` doc-comment URLs in `1116-*.js`; `t.I0.withTypes()`/`t.v9.withTypes()` = typed `useAppDispatch`/`useAppSelector` hooks, module `89959`). |
| Data fetching | Mix of Next.js **Server Actions** (encrypted action IDs via `(0,a.$)("<sha1-like hash>")`, module `58064`) for authenticated/sensitive calls (game launch, favorites, user profile), and client `fetch`/`next/image` for public assets. |
| i18n | `i18next` + `react-i18next`, locale resources compiled as webpack JSON modules per `./<locale>/<namespace>.json`, hydrated into RSC flight payload. |
| Styling | **Tailwind CSS** — arbitrary-value utility classes throughout (`h-[100vh] w-full m-0 p-0`, `w-8 h-8 border-4 rounded-full`), 8 separate CSS bundle files (per-route splitting). |
| Instant/crash games | **PixiJS** — `<link rel="preload" as="script" href="/assets/pixi-assets/js/pixi-legacy.min.js">` in every page `<head>`. Corroborated by ~35 dedicated i18n keys for an in-house Crash game (`crashGameTitle`, `autoCashout`, `jackpotDetails*`, `buttonTextStartAutoBet`, streak/wagered/profit stats) — this is a first-party WebGL/canvas crash game rendered client-side with Pixi, separate from third-party iframe slot titles. |
| Real-time | `socket.io-client` v3+ (module `6931-d699aa7dd30c7acb.js`, protocol markers `EIO=4`). Live connection observed going to `http://45.198.14.55:7032/api/socket/?EIO=4&transport=polling` — see §7 for the security implication of this. |
| Fonts | `next/font` self-hosted woff2, preloaded via `<link rel=preload as=font>`. |
| Analytics/monitoring | **None found** — no Sentry, no Google Analytics/GTM, no Meta Pixel, no Turnstile/reCAPTCHA/hCaptcha references anywhere in the downloaded bundles. Either analytics is server-side only, loaded from a page not yet crawled, or genuinely absent on this instance. |
| CDN/Edge | **Cloudflare** (`server: cloudflare`, `cf-ray`, `cf-cache-status: DYNAMIC`, NEL/report-to headers). |
| Image optimization | Next.js `/_next/image` (default loader, webp format, `contentSecurityPolicy: "script-src 'none'; frame-src 'none'; sandbox;"` scoped only to the image-optimizer response, not the app). |

---

## 4. Game Launch Flow

**Route pattern:**
```
/play-casino/[gameId]/[providerId]/[gameName]/[demoAvailable]
```
Confirmed live by clicking a real game tile in the lobby, which produced:
```
https://white-label-2.gammaplus.io/play-casino/9/19/Mad%20Buffalo/true
```
(`gameId=9`, `providerId=19`, `gameName="Mad Buffalo"`, `demoAvailable=true`).

There is also a 5th optional slug segment for tournament context, `tournamentId` (see code below).

**Client logic** (`chunks/page-83b6a91a0ee055ef.js`, deminified/reformatted):

```js
let { slug: j } = t;

// Resolve current user (via a server action bridge that reads the HttpOnly auth cookie
// server-side, since client JS cannot read it directly)
let L = async () => {
  let cookie = await getAuthCookie();          // (0,s.bW)()  — server action
  if (!cookie?.value) return null;
  let raw = await getUserProfile(cookie.value); // (0,o.ZP)() — server action
  let parsed = JSON.parse(raw);
  if (parsed?.data) { setUserData(parsed.data); return parsed.data; }
  return null;
};

// Actual game-launch call
let k = async (demo, walletId) => {
  let payload = { gameId: j?.[0], demo: demo || false, walletId };
  if (Number(j?.[4])) payload.tournamentId = Number(j[4]);
  console.log("Launching payload:", payload);

  let res = JSON.parse(await launchGame(payload));   // (0,i.pj)() — server action
  console.log("init game data:", res);

  let launchUrl = res?.data?.launchGameUrl
                || res?.data?.casinoGame?.launchGameUrl
                || "";

  if (res?.data?.type?.toUpperCase() === "HTML" || looksLikeHtml(launchUrl)) {
    // in-house / self-hosted HTML5 game: render as a Blob-URL iframe (bypasses direct HTML injection XSS by isolating in its own iframe + blob origin)
    setGameFrame({ type: "HTML", content: launchUrl });
    return;
  }
  if (res?.data?.type === "URL" && launchUrl?.startsWith("http")) {
    // third-party provider iframe: real remote URL
    setGameFrame({ type: "URL", url: launchUrl });
    return;
  }
  console.warn("Unknown game launch type");
};

// On mount: if slug[3] ("demoAvailable") === "true", show a Demo/Real chooser modal
// instead of auto-launching. Otherwise launch immediately with the resolved wallet id.
useEffect(() => {
  (async () => {
    if (!j?.[0]) return;
    let user = userDataRef.current || (await L());
    if (!user?.user?.wallets?.length) { console.log("❌ No wallet found."); return; }
    let walletId = selectedWallet?.value || user.user.wallets[0].id;
    if (j[3] === "true") { setShowDemoRealModal(true); return; }
    k(j[3], walletId);
  })();
}, [j]);

// Poll for balance refresh every 10s while the game is open
useEffect(() => {
  let interval = setInterval(() => {
    userDataRef.current?.user ? console.log("🔄 Refreshing user balance...") : console.warn("⚠️ userData disappeared → reloading");
    L();
  }, 10000);
  return () => clearInterval(interval);
}, []);
```

**HTML/Blob-embedded game player** (self-hosted games, same chunk):
```js
function HtmlGamePlayer({ htmlContent }) {
  let iframeRef = useRef(null);
  useEffect(() => {
    if (!htmlContent || !iframeRef.current) return;
    iframeRef.current.src = "about:blank";
    let blob = new Blob([htmlContent], { type: "text/html" });
    let url = URL.createObjectURL(blob);
    iframeRef.current.src = url;
    return () => URL.revokeObjectURL(url);
  }, [htmlContent]);

  useEffect(() => {
    let onMessage = async (e) => {
      if (e.source !== iframeRef.current?.contentWindow) return;
      let { type, data } = e.data;
      if (type === "UPDATE_BALANCE") {
        console.log("📢 Balance update received from iframe:", data);
        let cookie = await getAuthCookie();
        if (cookie?.value) {
          let raw = await getUserProfile(cookie.value);
          let parsed = JSON.parse(raw);
          if (parsed?.data) dispatch(setUserData(parsed.data));
        }
      }
      if (type === "GAME_EVENT") console.log("🎮 Game event:", data);
    };
    window.addEventListener("message", onMessage);
    return () => window.removeEventListener("message", onMessage);
  }, [dispatch]);

  // On load, push initial balance/user id into the game iframe
  useEffect(() => {
    iframeRef.current?.contentWindow?.postMessage({
      type: "INIT_GAME",
      data: { userBalance: wallets[0]?.balance || 0, userId: userData?.user?.id }
    }, "*");
  }, [/* ... */]);

  return <iframe ref={iframeRef} id="htmlGameContainer" title="game" />;
}
```

**`postMessage` protocol between host and embedded game:**

| Direction | Type | Payload |
|---|---|---|
| Host → game | `INIT_GAME` | `{ userBalance, userId }` |
| Game → host | `UPDATE_BALANCE` | arbitrary — triggers a full server-side balance re-fetch (host doesn't trust the game's number, it just re-pulls from the API) |
| Game → host | `GAME_EVENT` | logged only, no handling code found in this bundle |

**Navigation helper** for building the launch URL from a game-card object (`page-4a87a6f260719bfb.js` / `page-a5e36caee330b69f.js`, home & lobby):
```js
onGamePlayRedirect: (game) => {
  if (isAuthenticated) {
    router.push(`/play-casino/${game.uniqueId}/${game.providerId}/${game.name.EN}/${game.demoAvailable}`);
  } else {
    router.push("/login");
  }
},
generateProviderRoute: (providerId) => `/casino?provider=${providerId}`,
onRecentGamePlayRedirect: (game) => {
  if (isAuthenticated && game?.casinoProviderId && game.uniqueId) {
    router.push(`/play-casino/${game.uniqueId}/${game.casinoProviderId}/${game.name.EN}`);
  } else router.push("/login");
},
```

**Key architectural point:** game launch, favoriting, and user-profile lookups are **Next.js Server Actions**, not client-callable REST endpoints. The client bundle only ever contains an opaque action-id hash (e.g. `cab91b5d1b6cd2b4ed4202c840fe5cd2bc0bc684` for launch, `48cf1207d4fbf51bb742128e462f69fe1cfc481d` for reading the auth cookie, `d62dfa4bab86c45a9446529319c4933a0de2bd44` for favoriting); the browser POSTs to the *same page URL* with a `Next-Action` header, and the actual call to `white-label-api.gammaplus.io` happens **server-side inside the Next.js runtime**, where request headers/API keys never reach the browser. This was confirmed live: clicking "Demo" on the play-casino page fired a `POST https://white-label-2.gammaplus.io/play-casino/9/19/Mad%20Buffalo/true` (self-referential, no visible call to `white-label-api.gammaplus.io` in the Network panel).

**Live test result:** the demo launch did not render a playable game during this session — the backend origin behind `white-label-api.gammaplus.io` (or the socket layer specifically, see §7) is currently returning `503` for real-time features, and the game frame stayed blank before the SPA auto-redirected to `/en/bonus/all?bonusType=all`. This looks like a live backend/infra issue on this operator instance at the time of testing, not a client bug.

---

## 5. API Surface

### Hosts

| Host | Purpose | Source |
|---|---|---|
| `white-label-api.gammaplus.io` | Primary REST API base — `BASE_URL_1 = "https://white-label-api.gammaplus.io/api/v1/"` | Config object, module `20357` (inlined into `2622-*.js`, `layout-*.js`, `page-4a87a6f260719bfb.js`) |
| `white-label-api.gammaplus.io` | `SOCKET_URL` (declared) for socket.io | same config object |
| `white-label-chat.gammaplus.io` | Live chat widget, embedded as a same-origin-styled iframe (`<iframe id="chat-drawer" src="https://white-label-chat.gammaplus.io" title="Chat Drawer">`) on every page | `index.html`, live DOM |
| `gammastack-casino.s3.amazonaws.com` / `gammastack-casino.s3.us-east-1.amazonaws.com` | `IMAGE_URL` — CMS/banner image storage | config object, live banner requests |
| `agstatic.com` | Game thumbnail/box-art CDN for third-party provider titles (paths like `/games/rawgames/mad_buffalo.jpg`, `/games/avatarux/arcana_pop.jpg` — reveals provider aggregator slugs `rawgames`, `avatarux`) | live network capture |
| `http://3.220.85.69:8008/static/js/iframeRender.min.js` | `SPORTBOOK_SCRIPT_URL` — sportsbook iframe renderer script, loaded from a **raw IP over plain HTTP** | config object (see §7) |
| **`http://45.198.14.55:7032/api/socket/`** | **Actual runtime socket.io endpoint** observed live (polling transport), a raw IP:port over plain HTTP — differs from the `SOCKET_URL` declared in the bundled config | live network capture (see §7) |

### Config object (module `20357`, verbatim minified)
```js
{
  apiGateways: {
    BASE_URL_1: "https://white-label-api.gammaplus.io/api/v1/",
    BASE_URL_2: process.env.NEXT_AUTH_API_BASE_URL   // not inlined — server-only env var, empty in client bundle
  },
  SOCKET_URL: "https://white-label-api.gammaplus.io",
  IMAGE_URL: "https://gammastack-casino.s3.amazonaws.com/",
  SPORTBOOK_IFRAMEID: "1",
  SPORTBOOK_SCRIPT_URL: "http://3.220.85.69:8008/static/js/iframeRender.min.js",
  NEXT_PUBLIC_CHAT_URL: "https://white-label-chat.gammaplus.io"
}
```

### REST endpoint paths (client-visible literals)

Very few literal REST paths exist in the client bundle — as noted in §4, most authenticated calls (auth check, game launch, favorites, profile) go through **Server Actions**, whose real HTTP targets never reach the browser. The literal frontend *routes* (not API paths) found are already listed in §2. The one clearly server-only, non-Server-Action REST surface visible is the **socket.io namespace**: `GET/POST /api/socket/` (Engine.IO handshake, `EIO=4`), reachable both via the declared host (`white-label-api.gammaplus.io`) and observed in practice hitting a raw backend IP (§7).

### Server Actions inventory (encrypted action IDs found in the bundle, module `58064` wrapper)

| Exported as | Action hash | Purpose (inferred from call site) |
|---|---|---|
| `pj` | `cab91b5d1b6cd2b4ed4202c840fe5cd2bc0bc684` | Launch game (`gameId`, `demo`, `walletId`, optional `tournamentId`) → `{ data: { type: "HTML"\|"URL", launchGameUrl } }` |
| `bW` | `48cf1207d4fbf51bb742128e462f69fe1cfc481d` | Read auth token from HttpOnly cookie (server-side bridge) |
| `ZP` | `f8246ca2758a9684381f9c3b6e7ac2f8ee9d5357` | Fetch current user profile/wallets given the token |
| `kv` | `d62dfa4bab86c45a9446529319c4933a0de2bd44` | Toggle favorite game (`casinoGameId`, `userId`) |
| `pY` | (present, call site not fully traced) | Related to game listing / recent games |

### WebSocket / socket.io

- Client library: socket.io-client (Engine.IO v4 protocol), bundled in `6931-d699aa7dd30c7acb.js`.
- Purpose (inferred from chat iframe + balance-refresh pattern): likely balance/notification push and the chat widget's own transport (the chat iframe at `white-label-chat.gammaplus.io` runs its **own separate** socket.io client — console captured `Connection Error: OC: xhr poll error` from `white-label-chat.gammaplus.io/assets/index-D2quYBNB.js`, confirming the chat app is a **separate Vite-built SPA** embedded via iframe, not part of the Next.js bundle).
- Live behavior: polling handshake to `http://45.198.14.55:7032/api/socket/` returned **HTTP 503** during this test — backend/socket infra was degraded at analysis time.

### Endpoint groups NOT independently discoverable client-side (server-action-hidden)
Auth/register/login, KYC document upload, wallet deposit/withdraw, bonus claim, tournament enroll — all of these have rich i18n error-message coverage (§6) implying a full REST surface exists server-side, but the concrete paths are not present in any client bundle because they're invoked from Server Actions or from the Next.js server's own `fetch` calls during SSR, neither of which ship literal path strings to the browser.

---

## 6. Feature Inventory from i18n

All 8 locales (`en, fr, it, es, ja, nl, fi, de`; default `en`) ship **one single flat namespace called `"home"`**, not split by feature — it contains **1,796 keys** and is effectively the entire product's UI copy + validation/error message catalog. This is the most complete feature map available. Extracted to `retail/i18n/en-home-full.json`.

Notable branding leakage in the strings themselves: the footer copy identifies the platform vendor as **"Gammastack.com is owned and operated by HNA Gaming N.V."**, this specific white-label instance is skinned as **"Arc"/"Arc Casino"** (`<title>Arc</title>`, meta description `"Created and Owned by GS Core"`), yet one untranslated string still references a *different* brand name — `SupportedCurrencySpanText1: "We don't copy or adopt software from other websites. To make Tower.bet the..."` — a leftover from another GammaStack white-label customer's copy that wasn't overridden for this deployment. Evidence the translation/content layer is templated and can leak cross-tenant strings when an operator doesn't fully customize it.

### Namespace breakdown by feature area (keyword-matched, counts approximate since categories overlap)

| Area | Key count | Representative keys |
|---|---|---|
| Auth / Account | 316 | `login`, `registeraccount`, `2fa_description`, `select_2fa_method`, `accountMenuLogout`, `accountFrozen`, `accessTokenExpiredOrNotPassed` |
| Errors / Validation | 166 | `dailyWagerLimitMaxError`, `cpfError` (`"CPF must be of {{length}} characters"` — Brazil tax ID), `cnpjError`, `countryCodeRequired` |
| Wallet / Payments | 188 | `WalletSettingDisplayFiat` (crypto-in-fiat display toggle), `WalletSettingHideZero`, `INSTANTBANKING`, `DailyDepositLimitExceeded`, `amountDepositedSuccess` |
| Bonus / Promotions | 157 | `JoiningBonusButton`, `bonusAlreadyActive`, `amountToBeWagered`, `alreadyClaimedSpinWheelBonus` (daily roulette bonus wheel) |
| VIP / Loyalty | 66 | `loyaltyClub`, `CompletedAllTheVIPLevels`, "Level 6" unlock language, `loyaltyCashback`, `loyaltyPerCurrency` |
| Sports betting | 89 | `AutoBetFinished`, `BetPlacedSuccessfully`, `allBetsBetId`, `ProvablyFairMaximumBet` — shares vocabulary with the crash game (see below) |
| Crash / in-house instant games | 34 | `crashGameTitle`, `autoCashout`, `jackpotDetailsBettingRule`, `buttonTextStartAutoBet`, `timerTextFlewAway` ("Crashed") — this is the PixiJS-rendered title |
| Responsible Gambling | 84 | `depositLimitInfo`, `limitCantSetBefore`, `dailyBetLimitExceeded`, footer `Responsible Gambling` link |
| KYC / Verification | 52 | `UseShuftiProtoCompleteYourKYCVerificationInstantly` (confirms **Shufti Pro** as the KYC/identity-verification vendor), `ManualKYC`, `kycPendingDescription`, `generateKYCLink` |
| Tournament | 22 | `enrollTournamentSuccess`, `tournamentRebuyLimitReached`, `poolPrize`, `leaderBoard` |
| Referral / Affiliate | 34 | `myReferrals`, `referralReferAndEarn`, `referralEarn: " Earn ₹150 each"` (hardcoded INR amount — leftover from an India-market operator config), `enterReferralCode` |
| Chat / Support | 29 | `chatRule1`–`chatRule9` (community rules, one explicitly says `Do not insinuate ARC has bad intent ("scam site" etc)`), `faqSectionForgetPasswordDesc` |
| Notifications | 4 | minimal — likely a separate/undiscovered namespace or feature is thin |
| Games / Casino | 49 | `addToFavourite`, `gameCardTitle: "ARC Gaming Provably Fair Crypto Gambling Site"` (confirms provably-fair crypto positioning), `casinoTransactions` |
| CMS / Legal / Footer | 42 | `footerAboutSite` (copyright block naming HNA Gaming N.V.), `acceptPrivacyPolicy`, `bannerHeadTwo: "10SC Free Coins Await!"` ("SC" = sweepstakes coins — confirms **dual-currency sweepstakes model**, GC/SC, common in US social-casino-style operators) |

### Currency/market signals found in copy
- Sweepstakes coin model: `"10SC Free Coins Await!"`, `WalletSettingDisplayFiat` (crypto shown in fiat equivalent).
- Brazil-specific fields: `cpfError`, `cnpjError` (CPF/CNPJ tax IDs) — KYC form supports Brazilian identity documents.
- India-specific hardcoded amount: `referralEarn: " Earn ₹150 each"`.
- Crypto-first framing throughout (`gameCardTitle`, `buyCrypto`, `winRealCryptobyFreePlay`, `WalletSettingDisplayFiat`).

This confirms the GammaStack platform targets a broad, multi-market crypto/sweepstakes casino audience and the same translation catalog is reused (with residual un-scrubbed strings) across differently-branded operator deployments.

---

## 7. Security / Config Observations

1. **Raw backend IP + plaintext HTTP exposed to the client, twice:**
   - `SPORTBOOK_SCRIPT_URL: "http://3.220.85.69:8008/static/js/iframeRender.min.js"` — a third-party sportsbook rendering script is loaded over **unencrypted HTTP** from a bare IP address, shipped directly in client-readable config. This is both an MITM/script-injection risk (no TLS integrity) and an infra-fingerprinting leak.
   - Live socket.io traffic actually goes to `http://45.198.14.55:7032/api/socket/` — again a raw IP, plaintext HTTP, non-standard port — despite the bundled config declaring `SOCKET_URL: "https://white-label-api.gammaplus.io"`. This means either the app resolves the socket target dynamically at runtime (bypassing the CDN/TLS-terminating edge for the WebSocket upgrade) or there's a stale/dev value being hit. Either way, a real backend origin IP:port is directly reachable and observable by any player's browser dev tools, and it responded with **HTTP 503** during this test, suggesting it's a single origin with no CDN/WAF in front of it for this transport.
   - Recommendation for the operator: proxy sportsbook/socket traffic through the same Cloudflare-fronted TLS domain used for everything else, and never ship a bare IP in a publicly shipped bundle.

2. **No `Content-Security-Policy`, `Strict-Transport-Security`, or `X-Frame-Options` headers** on the main app responses (checked `/en/casino` and `/en/login` via `curl -I`). Given the app itself embeds third-party content via iframes (chat widget, potentially game iframes with `type: "URL"`) and injects arbitrary HTML into a Blob-URL iframe for "HTML"-type games, the absence of a CSP is a meaningful gap — a same-origin XSS would have no CSP backstop, and there's no explicit clickjacking protection (`X-Frame-Options`/`frame-ancestors`) at the app layer (Cloudflare may apply platform-level protections not visible in these headers, but nothing app-specific was observed).

3. **Auth token handling looks reasonably solid:** `curl` against unauthenticated pages sets **no cookies at all** (session/auth cookie is only issued after actual login), and in the live browser session `document.cookie` reads were blocked by the automation tooling's own privacy guard rather than exposing a token — consistent with the code using a Server Action (`bW`/`getAuthCookie`) specifically because the token is stored **HttpOnly** and inaccessible to client JS. Game launch and profile calls resolve the token server-side and never transmit it directly through client-visible fetches. This is good practice and mitigates the risk of the token leaking via XSS in the embedded game iframes.

4. **Server Actions as an API-hiding layer:** the deliberate choice to route `launch game`, `get profile`, `favorite game`, and `get auth cookie` through Next.js Server Actions (rather than a public `white-label-api.gammaplus.io` call made from the browser) means the real REST contract, and any API keys/headers needed to call it, never ship to the client. This significantly narrows this analysis's ability to enumerate the full backend REST surface (§5) — which is presumably the point.

5. **Public env var surface is minimal:** only `NEXT_PUBLIC_CHAT_URL` is inlined via `NEXT_PUBLIC_*` convention; `NEXT_AUTH_API_BASE_URL` (`BASE_URL_2`) is referenced via `process.env` but never resolves to a literal in the client bundle, confirming it's a **server-only** env var correctly excluded from the client build (no accidental secret leakage observed there).

6. **No CAPTCHA/bot-protection library** (Turnstile/reCAPTCHA/hCaptcha) found bundled — login/registration forms may rely purely on Cloudflare's edge-level bot management rather than an app-level challenge, or a challenge widget is loaded from a route not covered by this crawl (login/register page chunks were fetched but showed no captcha script references).

7. **Feature flags are page-level, not just UI-level:** routes like `/vip`, `/promotions`, `/live-casino`, `/providers`, `/responsible-gambling`, `/all` are compiled into the app (referenced in linked-to chunk code) but return **404** on this specific operator instance — confirming GammaStack's white-label product supports per-tenant enable/disable of entire page routes, not merely cosmetic per-tenant theming.

8. **Cross-tenant content leakage:** the `Tower.bet` brand name surfacing inside `SupportedCurrencySpanText1` on the "Arc"-branded instance (§6) demonstrates that shared/templated translation content isn't always fully re-authored per white-label customer, a minor but real brand-hygiene risk for GammaStack's operators.

---

## Appendix: File Inventory

```
retail/
├── index.html                  (1.1 MB, homepage SSR)
├── chunks/                     (41 files, 1.3 MB — JS chunks incl. layout2 route chunks)
├── css/                        (8 files, 404 KB)
├── manifest/                   (_buildManifest.js, _ssgManifest.js)
├── pages/                      (13 SSR HTML snapshots: casino, login, register, support,
│                                 bonus, tournaments, play-casino_test-game, + 6 404s for
│                                 disabled routes)
└── i18n/
    ├── en-home-full.json       (1,796-key English translation tree, primary feature map)
    └── 8x *.js                 (failed direct-chunk-fetch attempts, kept for reference —
                                  superseded by en-home-full.json)
```
