# White-Label Chat Application — Source Analysis

**Target:** `https://white-label-chat.gammaplus.io`
**Method:** Static reverse-engineering of the published Vite build (`index.html` + `/assets/*`), downloaded over HTTPS. No login, no form submission, no authenticated API or socket call, nothing modified. Cross-referenced against the already-captured player bundle in `src-analysis/retail/` and the admin endpoint inventory in `ADMIN-BUNDLE-ANALYSIS.md` §2.12.
**Analysis directory:** `/Users/artem/Projects/casino-management/src-analysis/chat/`

This is the third and last of the platform's front-end applications, and the only one that had never been looked at. It is architecturally the odd one out.

---

## 1. Download Summary

| Asset | File | Size |
|---|---|---|
| SPA shell | `index.html` | 430 B |
| Main bundle | `assets/index-D2quYBNB.js` | 859 KB |
| Lazy route chunk (the chat view) | `assets/Channel-CiZQLSUV.js` | 510 KB |
| Lazy route chunk (404 page) | `assets/NotFoundPage-C8ySTa47.js` | 519 B |
| CSS | `assets/index-DRQF-Gex.css` | 721 KB |
| Backgrounds / fallback image / notification sound | 6 files | 5.9 MB |
| Material Design Icons webfont | `assets/materialdesignicons-webfont-Dp5v-WZN.woff2` | 403 KB |
| Tier badges | `images/tier1..5.svg` | 11 KB |
| **Total** | **19 files** | **8.1 MB** |

`Last-Modified: Mon, 27 Jul 2026 11:52:44 GMT` on the main bundle — this build is four days old as of capture, so it is live and maintained, not an abandoned deployment.

**No source map is published.** `assets/index-D2quYBNB.js.map` returns HTTP 200, but the body is the 430-byte SPA `index.html` — the server is a catch-all that serves the shell for any unmatched path. The same applies to any probed route or non-existent asset, so HTTP status is useless for route discovery here; everything below comes from reading the bundle. Analysis was done against minified code.

The entire shell is:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8">
    <link rel="icon" href="/favicon.svg">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>GS Chat</title>
    <script type="module" crossorigin src="/assets/index-D2quYBNB.js"></script>
    <link rel="stylesheet" crossorigin href="/assets/index-DRQF-Gex.css">
  </head>
  <body>
    <div id="chat"></div>
  </body>
</html>
```

`GS Chat` = GammaStack Chat. Mount point `#chat`, not the conventional `#app`.

---

## 2. Tech Stack

**This app shares no framework with either of the other two.** The admin back office is React + Redux + CRA; the player site is Next.js App Router + React + Redux Toolkit. The chat app is **Vue 3 + Vuetify + Pinia + Vite**. Three apps, two ecosystems, three state-management libraries.

| Layer | Finding | Fingerprint evidence |
|---|---|---|
| Framework | **Vue 3** (Composition API, `<script setup>` SFCs) | `https://github.com/vuejs/core` and `https://link.vuejs.org/feature-flags` doc URLs in the bundle; `__name:` / `__file:` SFC metadata on every component; zero React markers (`ReactCurrentOwner`, `react-dom`: 0 hits) |
| Build tool | **Vite** | Asset naming `index-<hash>.js`, `crossorigin` module script, `vite:preloadError` event, `__vite__mapDeps` chunk-dependency table, `import.meta.env` → `VITE_APP_*` variables inlined |
| UI kit | **Vuetify 3.5.9** | `E2.version = O7`, `const O7 = "3.5.9"` verbatim; `VToolbarTitle`/`VSelectionControlGroup`/`VInfiniteScroll` component definitions; `vuetify-theme-stylesheet` injected `<style>` id |
| Icons | **Material Design Icons 7.4.47** | CSS `url(/assets/materialdesignicons-webfont-*.woff2?v=7.4.47)`; `mdi-alert-circle`, `mdi-dots-vertical`, `mdi-eye`/`mdi-eye-off` used in components |
| State | **Pinia** (with `pinia-plugin-persistedstate`-style `persist` option) | `https://pinia.vuejs.org/core-concepts/outside-component-usage.html`; every store declares `persist:{enabled, strategies:[{storage: localStorage, paths:[…]}]}` |
| Router | **vue-router 4** | `https://next.router.vuejs.org/guide/migration/#removed-star-or-catch-all-routes`; `createWebHistory` + `/:pathMatch(.*)*` catch-all |
| i18n | **vue-i18n 9** (`legacy:false`, Composition mode) | `vue-i18n-resource-inspector`, `vue-devtools-plugin-vue-i18n`; `SI({legacy:!1, locale, fallbackLocale, messages})` |
| HTTP | **axios**, one instance per declared service, with request/response interceptors | `xt.isAxiosError`, `xt.AxiosHeaders`, `e.interceptors.request.use(...)` |
| Real-time | **socket.io-client v3+** (Engine.IO v4) | `https://socket.io/docs/v3/migrating-from-2-x-to-3-0/` migration URL; `this.socket.binary`, `.on("packet")`, `.on("upgrade")` |
| Dates | **moment.js 2.30.1** — in the lazy chunk only | `g.version="2.30.1"` in `Channel-CiZQLSUV.js` |
| GIFs | **@giphy/js-fetch-api** | `https://github.com/Giphy/giphy-js/tree/master/packages/fetch-api`, `GIPHY_API_URL \|\| "https://api.giphy.com/v1/"` |
| Emoji | `emoji-datasource-apple@6.0.1` picker, images from **jsDelivr CDN** | `https://cdn.jsdelivr.net/npm/emoji-datasource-apple@6.0.1/img/apple/64` |
| Audio | **howler.js** | `new eb.Howl({src:[Kk]})` where `Kk = "/assets/messageReceive-CtOLJ-sq.mp3"` |
| Clipboard | **clipboard.js** | `https://clipboardjs.com/` |
| CDN/Edge | **Cloudflare** | `server: cloudflare`, `cf-ray`, `cf-cache-status` |

### Build provenance leaked in full

Vue SFC compiler output retains `__file` paths. The **complete original source tree** is recoverable:

```
/home/node/app/src/
├── App.vue
├── layouts/{ChatLayout,DefaultLayout}.vue
├── views/pages/Channel.vue
├── components/
│   ├── chatheader/index.vue
│   │   └── modals/{ChatRules,ChatSettings}.vue
│   ├── chatbody/index.vue
│   │   ├── components/{ChatMessage,EventDescription,ReplyMessage,ShareEvent,TipMessage,UserInfoIcon}.vue
│   │   │   └── eventTypes/{Achievement,Bets,Bonus,CommonTemplate}.vue
│   │   └── modals/{BlockUser,PlayerStatistics,TipPlayer}.vue
│   ├── chatfooter/index.vue
│   │   └── components/{ChannelBan,ChatDisabledError,CommonError,FooterError,Gif,
│   │                   KycError,MessageInput,PermanentBanError,RankingLevelError}.vue
│   ├── chatrain/index.vue
│   │   └── components/ChatRainMessage.vue
│   └── common/{CustomButton,CustomDialog,CustomInput,CustomPageLoader,CustomSlider,Snackbar}.vue
└── assets/icons/*.vue  (14 inline SVG components)
```

`/home/node/app` is a Docker `node` image working directory — the app is containerised and built inside the image.

### Template residue

The `ja` locale bundle still carries the full navigation tree of the **Vuetify admin starter template** the project was scaffolded from: `ecommerceCart`, `usersList`, `authRegister`, `errorNotFound`, `lottieAnimation`, `waterFall`, and brand names `Nitori` / `IKEA` / `Unsplash` / `Booking`. None of it is reachable in this app. Same class of finding as the `Tower.bet` string in the player bundle (`RETAIL-SOURCE-ANALYSIS.md` §6) — boilerplate that was never scrubbed.

---

## 3. Routes

Three routes, defined in the main bundle:

| Path | Name | Layout | Meta | Component |
|---|---|---|---|---|
| `/` | `global_channel` | `chat` | `isFullscreen: false` | `Channel.vue` (lazy) |
| `/chat-fullscreen` | `chat_fullscreen` | `chat` | `isFullscreen: true` | `Channel.vue` (lazy, same component) |
| `/:pathMatch(.*)*` | `error` | (default) | — | `NotFoundPage` (lazy) |

`createWebHistory()`, so these are real paths, not hashes. `isFullscreen` is read throughout the component tree (`route.meta.isFullscreen`) to switch the emoji picker, GIF picker and settings modals between a dropdown and a full-page presentation — i.e. `/chat-fullscreen` is the mobile/expanded rendering of the identical channel view.

There is **no per-channel route**. Channel selection is in-memory state (the `channel` Pinia store), driven by a dropdown in the header. The URL never changes when you switch channels.

---

## 4. Authentication — the interesting part

The chat runs on a **different origin** from the player site (`white-label-chat.gammaplus.io` vs `white-label-2.gammaplus.io`), embedded as `<iframe id="chat-drawer">`. It therefore cannot read the player site's cookies. The platform solves this with a **`postMessage` token handover**, and the design has a real consequence.

### 4.1 The handshake, both sides

**Parent → child.** `src-analysis/retail/chunks/layout-180563d97be75b95.js`, webpack module `23074` (deminified):

```js
export default function ChatPanel({ authToken }) {
  const dispatch = useAppDispatch();
  const { push } = useRouter();
  const { isChatOpen } = useAppSelector(s => s.general);

  useEffect(() => {
    window.addEventListener("message", e => {
      if (e.origin === config.NEXT_PUBLIC_CHAT_URL) {          // exact-match origin check
        const msg = safeJsonParse(e.data);
        if (msg?.action === CHAT_ACTIONS.CLOSE_CHAT)   dispatch(setIsChatOpen(false));
        if (msg?.action === CHAT_ACTIONS.LOGIN_PROMPT) push("/login");
      }
    });

    timer = setTimeout(() => {
      postSetTenant();                                          // { action: "setTenant" }
      authToken ? postLogin(authToken)                          // { accessToken, action: "addToken" }
                : postLogout();                                 // { action: "removeToken" }
    }, 1000);

    return () => { /* … */ clearTimeout(timer); };
  }, []);

  return (
    <div className={`h-full m-0 p-0 chat-panel z-[99] ${isChatOpen ? "w-[356px]" : "w-[0]"}`}>
      <iframe src={config.NEXT_PUBLIC_CHAT_URL} id="chat-drawer" className="h-[100vh] w-full m-0 p-0" title="Chat Drawer" />
    </div>
  );
}
```

and the senders (same chunk, and duplicated in `2622-*.js` / `8110-*.js` / `page-4a87a6f260719bfb.js`):

```js
const postLogin  = t => document.getElementById("chat-drawer")?.contentWindow
    .postMessage(JSON.stringify({ accessToken: t, action: ACTIONS.LOGIN }),  config.NEXT_PUBLIC_CHAT_URL + "/");
const postLogout = () => …postMessage(JSON.stringify({ action: ACTIONS.LOGOUT }),                config.NEXT_PUBLIC_CHAT_URL + "/");
const postSetTenant = () => …postMessage(JSON.stringify({ action: ACTIONS.SET_TENANT }),         config.NEXT_PUBLIC_CHAT_URL + "/");
const postSetTheme = e => …postMessage(JSON.stringify({ action: ACTIONS.SET_THEME, layoutName: e }), config.NEXT_PUBLIC_CHAT_URL + "/");
```

**Child → parent.** `App.vue` in the chat bundle:

```js
onMounted(() => {
  if (storedToken) general.setAuthToken(storedToken);
  loadTheme("default");

  window.addEventListener("message", e => {
    if (!CONFIG.VITE_APP_TARGET_URLS?.split(",").includes(e.origin)) return;   // origin allowlist
    if (typeof e.data !== "string") return;
    const msg = JSON.parse(e.data);

    if (msg?.accessToken && msg?.action === ACTIONS.LOGIN) {
      setAccessToken(msg.accessToken);        // → localStorage["accessToken"]
      general.setAuthToken(msg.accessToken);
      auth.setIsLoggedIn(true);
      window.location.reload();
    } else if (msg?.action === ACTIONS.LOGOUT) {
      removeAccessToken(); general.setAuthToken(null); auth.setIsLoggedIn(false);
      window.location.reload();
    } else if (msg?.action === ACTIONS.SET_TENANT) {
      setTenantUrl(e.origin);                 // → localStorage["tenantUrl"]
    } else if (msg?.action === ACTIONS.SET_THEME) {
      loadTheme(msg.layoutName);
    }
  });
});
```

and the outbound helper:

```js
const postToParent = e => {
  const ancestor = window.location.ancestorOrigins?.[0];
  if (ancestor && window.top) window.top.postMessage(JSON.stringify(e), `${ancestor}/`);
};
const promptLogin = () => postToParent({ action: ACTIONS.LOGIN_PROMPT });
const closeChat   = () => postToParent({ action: ACTIONS.CLOSE_CHAT });
```

### 4.2 Message protocol

| Action constant | Wire value | Direction | Payload | Handled by parent? |
|---|---|---|---|---|
| `LOGIN` | `addToken` | parent → chat | `{accessToken, action}` | n/a |
| `LOGOUT` | `removeToken` | parent → chat | `{action}` | n/a |
| `SET_TENANT` | `setTenant` | parent → chat | `{action}` | n/a |
| `SET_THEME` | `setTheme` | parent → chat | `{action, layoutName}` | n/a |
| `LOGIN_PROMPT` | `showLoginPrompt` | chat → parent | `{action}` | yes → `router.push("/login")` |
| `CLOSE_CHAT` | `closeChat` | chat → parent | `{action}` | yes → `setIsChatOpen(false)` |
| `REDIRECT` | `redirect` | chat → parent | `{action, redirectUrl}` | **no — silently dropped** |
| `TERMS_AND_CONDITIONS` | `termsAndConditions` | chat → parent | `{action}` | **no — silently dropped** |

The chat's action enum has eight members; the player site's has six. `REDIRECT` and `TERMS_AND_CONDITIONS` exist only on the chat side. `REDIRECT` is fired when a player taps the "Big Win Alert" achievement card (`eventTypes/Achievement.vue` → `postToParent({redirectUrl: event.eventURL, action: REDIRECT})`), and `TERMS_AND_CONDITIONS` from the chat-rules dialog. Neither does anything on this tenant — **the two apps are deployed at different versions of the same contract.** Observed, not inferred: both enums are verbatim in their respective bundles.

### 4.3 The origin allowlist does not contain the parent

The chat's build-time config, verbatim from `assets/index-D2quYBNB.js`:

```js
const CONFIG = {
  apiGateways: { VITE_APP_BASE_URL_1: "http://45.198.14.55:7032" },
  SOCKET_URL: "http://45.198.14.55:7032",
  VITE_APP_TARGET_URLS: "http://45.198.14.55:8024,http://103.119.170.165:8022,http://45.198.14.55:7032"
};
```

The allowlist is three bare-IP HTTP origins. The actual parent frame on this tenant is `https://white-label-2.gammaplus.io`, which is **not in the list**. The `App.vue` listener therefore returns early on every message the player site sends, including `addToken`.

**Consequence:** on this deployment the chat iframe can never receive the player's token by the intended path, so it can only ever operate in the anonymous/global-channel mode. This is consistent with what the previous session observed live — the chat iframe throwing `Connection Error: xhr poll error` from `white-label-chat.gammaplus.io/assets/index-D2quYBNB.js` — and with the 503s from the socket host. Whether the root cause is the origin mismatch or the backend being down cannot be separated without an authenticated session, which is out of scope here; the mismatch is a fact of the bundle either way.

**The architectural point matters more than the bug.** `VITE_APP_TARGET_URLS` is a Vite build-time variable — Vite inlines `import.meta.env.*` as literals at compile time. The set of origins allowed to hand a token to the chat is baked into the JavaScript. So **every white-label tenant needs its own rebuild and its own deployment of the chat app**, or the operator must widen the allowlist to cover all tenants, which would let any listed tenant's page inject a token into any other's chat frame. For a product whose entire premise is multi-tenancy, this is the weakest seam found across the three bundles. The player site does this correctly by contrast — `NEXT_PUBLIC_CHAT_URL` is read at runtime.

### 4.4 The token is deliberately HttpOnly on one origin and in `localStorage` on the other

`RETAIL-SOURCE-ANALYSIS.md` §7.3 credited the platform for keeping the player session token in an HttpOnly cookie, unreachable from client JS, with a Server Action (`getAuthCookie`) as the only bridge. That holds for the player origin. The chat integration then routes around it:

1. The Next.js server reads the HttpOnly cookie and passes it as the `authToken` **prop** to the `ChatPanel` client component. Being a client-component prop, it is serialised into the RSC flight payload and is therefore present in the page's own HTML. This is directly visible in the captured homepage — `src-analysis/retail/index.html` contains, in a `self.__next_f.push` chunk:

   ```
   ["$","$L24",null,{"authToken":"$undefined"}]
   ```

   `$L24` is the lazily-referenced `ChatPanel`, and `$undefined` is the flight encoding for `undefined` because the capture was unauthenticated. The prop slot is real; on a logged-in page the session token occupies it as a plain string in the served HTML. *(The structure is observed; the populated value was not — no authenticated page was fetched.)*
2. `ChatPanel` `postMessage`s it to a **different origin**.
3. The chat app writes it to `localStorage["accessToken"]` (`setAccessToken = e => localStorage.setItem("accessToken", JSON.stringify(e))`), where it persists across sessions and is readable by any script on the chat origin.
4. Every subsequent chat REST call and socket handshake carries it (§5, §6).

So the session token's real exposure is set by the weakest of the two origins, and that is the chat app: `localStorage`, no CSP, no HSTS, over plaintext HTTP to a bare IP. The HttpOnly cookie on the player origin narrows the blast radius of an XSS there but does not describe where the token actually lives.

### 4.5 Tenant id is hardcoded

```js
const getTenantId = () => 1;
```

Verbatim. It is called in `App.vue`, the chat header, and the channel view, and its value is sent as the `tenant` socket handshake header and as the `tenantId` query parameter on the Chat Rain fetch. The `general` Pinia store has a real `tenantId` field with a `setTenantId` action and `persist.paths:["tenantId"]`, and the `SET_TENANT` postMessage action exists — but `SET_TENANT` writes the parent origin to `localStorage["tenantUrl"]` and never calls `setTenantId`. The store field is dead; the constant `1` is what ships. Multi-tenancy in this app is currently notional.

---

## 5. REST API Surface

**Base URL:** `http://45.198.14.55:7032/api/v1` — composed as `CONFIG.apiGateways.VITE_APP_BASE_URL_1 + "/api/v1"`. A **bare IPv4 address over plaintext HTTP on a non-standard port**, shipped in a public bundle. This is the same host:port the previous session saw the player site's socket.io polling hit (`RETAIL-SOURCE-ANALYSIS.md` §5/§7). The chat app uses it for *all* traffic, REST included — it never touches `white-label-api.gammaplus.io`.

One axios instance is created per entry in the service map, with `Content-Type: application/json`, `Accept: application/json`.

### 5.1 Endpoint registry

Verbatim from the bundle (`qt` object), with the HTTP verb taken from each call site:

| Method | Path (relative to `/api/v1`) | Purpose | Notes |
|---|---|---|---|
| GET | `/live-chat/get-chat-group` | List channels | → `{groupDetails:[…]}` |
| POST | `/live-chat/join-chat-group` | Join a channel | body `{chatGroupId}` → `{groupJoined}` |
| GET | `/live-chat/get-chat` | Message history for a channel | → `{chatDetails:[…], count}` |
| GET | `/live-chat/get-global-group-chat` | Message history for the global channel | → `{rows:[…], count}` — **different envelope from the above** |
| GET | `/live-chat/theme` | Per-tenant theme object | params `{key}` → `{theme}`, applied to Vuetify's dark theme |
| POST | `/live-chat/tip-to-chat` | Player-to-player tip | see §7.2 |
| GET | `/live-chat/get-chat-rain` | Active Chat Rain drops | params `{tenantId, groupId}` → `{chatRainDetails:[…]}` |
| POST | `/live-chat/claim-chat-rain` | Claim a Chat Rain | body `{userId, chatRainId, tenantId, groupId}` |
| GET | `/user/get-user` | Current user + wallets + chat settings | → `{user}` |
| GET | `/user/user-detail/{playerId}` | Another player's public stats | path param appended |
| GET | `/user/list` | Player list for @-mention autocomplete | → `{allUsers}` |
| GET | `/user/get-balance` | Wallet balance | |
| GET | `/user/get-chat-setting` | Tenant-default chat settings (anonymous users) | |
| PUT | `/user/update-user-setting` | Save the player's chat settings | |
| GET | `/report-user/get` | Players this user has blocked | → `{reportedUser}` |
| POST | `/report-user/block` | Block a player | |
| POST | `/report-user/unblock` | Unblock a player | |
| GET | `/common/get-currencies` | Currency list (id, code, name, symbol) | `handlerEnabled:false` → unauthenticated |

### 5.2 Cross-reference with the admin side

`ADMIN-BUNDLE-ANALYSIS.md` §2.12 lists the operator-facing half under `/api/v2/live-chat/…`. They line up as a clean read/write split across two services:

| Concern | Player side (`api/v1`, this app) | Operator side (`api/v2`, admin) |
|---|---|---|
| Channels | `get-chat-group`, `join-chat-group` | `get-group`, `create-group`, `update-group`, `delete-group`, `all-group`, `get-group-users` |
| Messages | `get-chat`, `get-global-group-chat`, socket `send-message` | `get-group-chats`, `messages/messages-group` |
| Chat Rain | `get-chat-rain`, `claim-chat-rain` | `get-chat-rain`, `create-chat-rain`, `update-chat-rain`, `delete-chat-rain` |
| Moderation | `report-user/block`, `report-user/unblock` (player-level mute) | `ban-group-user` (operator ban) |
| Offensive words | **nothing** | `get-offensive-words`, `create-offensive-word`, `update-offensive-word`, `delete-offensive-word` |

The last row is the notable one. The admin has a full CRUD screen for an offensive-word list; the chat client has **no client-side filter at all**. Searched `assets/*.js` for `offensive`, `profan`, `badword`, `blacklist`, `censor`, `filterWord` — the single case-insensitive hit is the phrase "using offensive language" inside the human-readable chat-rules copy. **Profanity filtering is entirely server-side**, applied on the `send-message` socket event. That is the right place for it (a client-side filter is trivially bypassed), and it is worth recording as a deliberate correct choice rather than an omission.

### 5.3 Auth and response envelope

Request interceptor:

```js
const onRequest = cfg => {
  if (cfg?.handlerEnabled) {
    const token = getAccessToken();                              // localStorage["accessToken"]
    if (token) cfg.headers.Authorization = `AccessToken=${token}`;
  }
  if (cfg?.loader) loaderStore.stopLoader();
  return cfg;
};
```

The scheme is **`Authorization: AccessToken=<token>`** — not `Bearer`, and not even RFC 7235 shape (`scheme SP credentials`, no `=`). A custom parser must sit on the server. `handlerEnabled` defaults to `true`, so every endpoint except `/common/get-currencies` is authenticated.

Response interceptor unwraps `response.data.data`, so the envelope is `{ data: <payload> }` on success and `{ errors: [{ description, … }] }` on failure. Status handling:

| Status | Behaviour |
|---|---|
| 400 / 403 / 500 | snackbar shows `errors[0].description`, promise rejected |
| **401** | **clears the stored token and `window.location.reload()`**, then shows the error |
| 404 | snackbar shows `statusText` only |
| 409 | snackbar shows `errors[0].description`, promise **not** rejected |

The 401 path is worth flagging: any 401 from any endpoint wipes the token and hard-reloads. Since the token can only be re-obtained via a `postMessage` from the parent — which on this tenant is blocked by the origin allowlist (§4.3) — a single 401 logs the chat out permanently for that page load.

---

## 6. Socket.io Protocol

The most complete real-time contract recovered from any of the three bundles.

### 6.1 Connection

```js
const useSocket = (namespace, handlers, extraHeaders) => {
  const connect = () => {
    const socket = io(`${CONFIG.SOCKET_URL}/${namespace}`, {
      path: "/api/socket",
      extraHeaders
    });
    socket.on("connect", onConnect);
    handlers?.forEach(({ eventName, handleData }) => socket.on(`/${namespace}/${eventName}`, handleData));
    socket.on("connect_error", onError);
    socket.on("disconnect", onDisconnect);
    registry[namespace] = { connected: false, socket, error: "" };
    window.privateSockets = registry;                 // ← global
  };
  …
};
```

| Property | Value |
|---|---|
| Endpoint | `http://45.198.14.55:7032/user-chat` |
| Engine.IO path | `/api/socket` |
| Namespace | `user-chat` (the only one; `NAMESPACES = {userChat: "user-chat"}`) |
| Transport | default (polling → websocket upgrade) |
| Reconnect | on `disconnect` with reason `"io server disconnect"`, close and re-`connect()` manually |

### 6.2 Room selection happens in HTTP headers, not by an emit

```js
useSocket(
  NAMESPACES.userChat,
  [ /* handlers, §6.3 */ ],
  token
    ? { "access-token": token, tenant: tenantId, group: selectedChannel?.id }
    : {                        tenant: tenantId, group: selectedChannel?.id }
);
```

There is no `join` emit. **Authentication, tenant and room are all carried as Engine.IO handshake `extraHeaders`**, and the subscription is re-established by tearing down and rebuilding the socket whenever the watched tuple `[token, selectedChannel.isJoined, selectedChannel]` changes. Switching channels means a full socket reconnect.

Two things follow. First, `extraHeaders` in socket.io-client only applies to the **polling** transport — the WebSocket handshake in a browser cannot set arbitrary headers — so either the server pins this connection to long-polling, or it reads the room from the session established during the polling handshake before upgrade. Second, the header name is `access-token` here, while REST uses `Authorization: AccessToken=…`. Two different conventions for the same credential in the same app.

This matches the player site's own socket layer, which uses the same `path: "/api/socket"` and the same `extraHeaders: {"access-token": token}` convention against namespaces `public` / `private` for `wallet`, `notification` and `dispute` events (`src-analysis/retail/chunks/layout-180563d97be75b95.js`, module `98335`). Same house style, separate connections.

### 6.3 Events

Event names are composed as `` `/${namespace}${EVENT}` `` for emits and `` `/${namespace}/${eventName}` `` for subscriptions.

**Inbound (server → client)** — `EVENTS = {LIVE_CHAT:"liveUserChat", CHAT_RAIN:"liveChatRain", CLOSED_CHAT_RAIN:"closedChatRain", DELETE_CHAT_RAIN:"deleteChatRain", UPDATE_CHAT_RAIN:"updateChatRain"}`:

| Wire event | Payload | Client effect |
|---|---|---|
| `/user-chat/liveUserChat` | `{data: <message>}` | append to `chatStore.chats`; if scrolled up and not own message, bump unread counter; play sound per notification setting; autoscroll if own message or at bottom |
| `/user-chat/liveChatRain` | `{data: <chatRain>}` | append to `chatRainDetails` (`key:"newChatRain"`) |
| `/user-chat/closedChatRain` | `{data: {id}}` | remove by id (`key:"closeChatRain"`) |
| `/user-chat/updateChatRain` | `{data: <chatRain>}` | `Object.assign` onto the matching entry by id |
| `/user-chat/deleteChatRain` | `{data: {id}}` | remove by id — mapped to the same `closeChatRain` reducer branch |

**Outbound (client → server)** — one event only, `SEND_MESSAGE = "/send-message"`, emitted as `/user-chat/send-message` with an empty-string second argument (no ack callback):

```js
// text
socket.emit("/user-chat/send-message", {
  chatGroupId: selectedChannel?.id,
  message: encodeMentions(input),          // see §7.1
  messageType: "MESSAGE",
  ...(replyMessage && { replyMessageId: replyMessage.messageId })
}, "");

// gif
socket.emit("/user-chat/send-message", {
  chatGroupId: selectedChannel?.id,
  gif: gifUrl,
  messageType: "GIF"
}, "");
```

Tips and Chat Rain claims go over **REST**, not the socket; the resulting `TIP` / `CHAT_RAIN` messages arrive back over `liveUserChat`.

### 6.4 Message shape

Assembled from every field dereferenced in `ChatMessage.vue`, `ReplyMessage.vue`, `TipMessage.vue`, `ShareEvent.vue`, `ChatRainMessage.vue` and the four `eventTypes/*.vue`:

```jsonc
{
  "messageId":   123,
  "chatGroupId": 4,
  "userId":      65,
  "user":        { "id": 65, "username": "player1", "profilePicture": "…" },
  "messageType": "MESSAGE | GIF | EVENT | TIP | CHAT_RAIN",
  "message":     "hello @{player2:66}",       // MESSAGE only
  "gif":         "https://media.giphy.com/…", // GIF only
  "createdAt":   "2026-07-27T11:52:44.000Z",

  "replyMessage": { /* nested message, rendered in ReplyMessage.vue */ },

  // messageType === TIP
  "tip":           { "amount": 5.00, "currency": "USD" },
  "recipientUser": { "username": "player2" },

  // messageType === EVENT | CHAT_RAIN
  "sharedEvent": {
    "eventType":        "Achievement | Tournament | Bet | Bonus",
    "eventName":        "…",
    "eventTitle":       "…",
    "eventDescription": "…",
    "eventImage":       "…",          // falls back to /assets/event_image_fallback-*.webp on error
    "eventURL":         "…",          // Achievement only → REDIRECT postMessage
    "eventAmount":      "…",
    "amountCurrency":   "…",
    "winAmount":        "…",          // Bet
    "gameName":         "…",          // Bet
    "currencyCode":     "USD"
  }
}
```

`EVENT` messages are the social-proof feed: a player's big win, tournament placing, achievement or bonus is auto-posted into the channel as a rich card. Four renderers, dispatched on `sharedEvent.eventType` — `Achievement` (tappable, redirects the parent frame), `Bets` ("Big Win Landed!", game art + win amount), `Bonus` (title over a background image), and `CommonTemplate` as the fallback, which is what `Tournament` actually renders through.

### 6.5 Chat Rain

`chatRain` entity, from `chatrain/index.vue`:

```jsonc
{
  "id": 12, "name": "Friday Rain", "prizeMoney": 100,
  "currencyId": 1, "currency": 1,
  "validFrom": "2026-07-27T12:00:00Z",
  "validFor":  15,                 // minutes
  "tenantId": 1, "chatGroupId": 4
}
```

The client resolves the currency symbol by matching `currencyId` against the `/common/get-currencies` list (falling back to the `¤` generic-currency sign), then runs a 1-second `setInterval` counting down to `moment.utc(validFrom).add(validFor, "minutes")`, hiding the card at zero. Claiming posts `{userId, chatRainId, tenantId, groupId}` to `/live-chat/claim-chat-rain`; an anonymous click fires `LOGIN_PROMPT` to the parent instead.

This is the player-facing half of the admin's `create-chat-rain` / `update-chat-rain` / `delete-chat-rain` screens — an operator-scheduled prize drop into a chat channel, with a claim window measured in minutes. A retention mechanic, and it is genuinely a distinguishing feature: it needs the chat, the wallet and the scheduler to be one system.

---

## 7. Feature Inventory

### 7.1 Messaging

- **160-character limit**, enforced client-side only, shown as a live countdown (`160 - input.length`).
- **@-mentions**: the composer keeps a `taggedPlayersList` and, on send, rewrites each `@username` to the wire form `@{username:userId}` via `new RegExp("@" + username + "\\b", "g")`. On render the inverse regex `/@{([^:]+):[^}]+}/g` turns it back into a highlighted span. Autocomplete is fed by `GET /user/list`.
- **Reply threading**: `replyMessageId` on send, nested `replyMessage` on receipt.
- **GIFs** via Giphy trending/search, infinite-scrolled 20 at a time.
- **Emoji** picker using `emoji-datasource-apple@6.0.1`, images fetched from jsDelivr.
- **Notification sound** (`messageReceive.mp3` via howler), gated on the player's `notificationSound` setting — `all`, `only_mentions` (matched by testing whether the message contains the player's own `@{username:id}` token), or `nothing`.
- Per-message actions: copy message, copy nickname, reply, tip, view profile.
- Unread-message counter while scrolled up; history paginated by infinite scroll.

### 7.2 Player-to-player tips

`TipPlayer.vue`. Amount + currency + **account password**, submitted to `POST /live-chat/tip-to-chat`:

```js
await tipPlayer({
  amount:       Number(amount || 0),
  recipientId:  parseInt(props.playerId),
  currencyCode: selectedCurrency?.value,
  chatGroupId:  props.chatGroupId,
  password:     Buffer.from(passwordInput.value).toString("base64")
});
```

See §8.2 — this is the most serious finding in the app.

Client-side validation only: minimum 0.1 (`"Negative values not allowed"` — the message does not match the rule), and a balance check against `user.wallets.find(w => w.currency.code === selected).amount` yielding `"Insufficient Funds"`. Currency defaults to `currencies[0]`. The tip renders in-channel as a `TIP` message ("Sharing the luck — $5.00 shared to player2").

### 7.3 Moderation and access gating

Two distinct concepts, both present:

- **Player-level blocking** — `report-user/block` / `unblock`, with a managed block list. Personal mute.
- **Operator-level restriction** — five gate types, `RESTRICTIONS = {kycCriteria:"KYCSTATUS_NOT_SUPPORTED", rankingLevelCriteria:"RANKING_LEVEL_CRITERIA", permanentBan:"PERMANENT_BAN", chatDisabled:"CHAT_DISABLED", channelBan:"BANNED_FROM_ADMIN_KINDLY_CONTACT_SUPPORT"}`.

`FooterError.vue` replaces the composer with the matching explanation component:

| Constant | Component | Trigger |
|---|---|---|
| `KYCSTATUS_NOT_SUPPORTED` | `KycError` | channel requires a verified KYC status — offers "Verify KYC" |
| `RANKING_LEVEL_CRITERIA` | `RankingLevelError` | channel requires a minimum VIP/loyalty tier |
| `PERMANENT_BAN` | `PermanentBanError` | `user.isBannedChatPermanently === true` |
| `CHAT_DISABLED` | `ChatDisabledError` | `defaultChatSettings.chat === "disable"` — tenant-wide kill switch |
| `BANNED_FROM_ADMIN_KINDLY_CONTACT_SUPPORT` | `ChannelBan` | per-channel ban, matches the admin's `ban-group-user` |
| *(any)* | `CommonError` | renders `channel.restrictions.description` as an "Eligibility Conditions" list |

Channels carry `{id, name, isGlobal, isJoined, restrictions: {isRestricted, reason: [{key, …}], description: [{<heading>: [<condition>, …]}]}}`. **The eligibility rules are server-authored and rendered generically** — the client displays whatever headings and bullet lists the API returns. So an operator can gate a channel on arbitrary criteria without a client release, which is the right shape for a white-label product.

Anonymous users may read `isGlobal` channels; clicking any non-global channel raises `LOGIN_CRITERIA` and prompts login through the parent frame.

### 7.4 Player profile card

`PlayerStatistics.vue` (`GET /user/user-detail/{playerId}`) shows username, avatar, tier badge (`/images/tier1..5.svg`, five loyalty tiers), and statistics/top-games sections — i18n keys `statistics`, `details`, `top`, `games`, `wager`, `tier`. Reachable by tapping any username. Also the entry point for block and tip.

### 7.5 Chat settings

`{ displayGIF: boolean, displayLevel: boolean, fontSize: 8–18 step 2, notificationSound: "all" | "only_mentions" | "nothing" }`, `PUT /user/update-user-setting`, with a "Defaults" button that restores the tenant defaults from `/user/get-chat-setting`. Anonymous users get the tenant defaults read-only. The `displayLevel` / "Display My Tier Level" switch is **commented out in the source** — the state field and the translation key ship, the control does not.

### 7.6 Theming

`GET /live-chat/theme?key=default` returns `{theme}`, assigned into Vuetify's dark theme at runtime (`vuetifyTheme.themes.value.dark = themeObject.theme`). The parent can switch it later with `{action:"setTheme", layoutName}`, which re-fetches with that key. So chat theming is server-driven per tenant and coupled to the player site's `layout1`/`layout2` skin selector documented in `RETAIL-SOURCE-ANALYSIS.md` §2. Unlike the origin allowlist, this part of tenancy is done at runtime and correctly.

---

## 8. i18n

**Two locales: `en` and `ja`.** Both are compiled into the main bundle — no lazy loading, no runtime fetch, no translation-management integration.

```js
const AVAILABLE = [
  { code: "en", flag: "us", name: "united-states", label: "English",  messages: EN },
  { code: "ja", flag: "jp", name: "japan",         label: "日本語",    messages: JA }
];
let initial = "en";
try { const [lang] = navigator.language.split("-"); if (["en","ja"].includes(lang)) initial = lang; } catch (e) { console.log(e); }
```

Locale is chosen from `navigator.language` at startup. There is **no locale switcher in the UI**, and the parent frame has no postMessage action to set one.

**This is a genuine white-label gap.** The player site ships **eight** locales (`en, fr, it, es, ja, nl, fi, de`) with 1,796 keys each, and the admin has a `/languages-management` screen for editing translations without a deploy. The chat supports two, hardcoded at build time, and only the `chat.*` namespace (~90 keys) is real product copy — the rest of `ja` is untranslated starter-template navigation (§2). A French or German player on this platform gets an English chat panel inside a fully localised site, and no operator-side tooling can change that.

Also hardcoded rather than translated: the currency symbol map `{USD:"$", INR:"Rs", SGD:"S$", LTCT:"Lt"}`. Four currencies, in code, with no fallback in the components that use it (`TipMessage.vue`, `Bets.vue`, `ChatRainMessage.vue` all render `SYMBOLS[code]` directly, so any other currency renders `undefined`). The Chat Rain card is the only place with a fallback (`¤`), and it takes a different route — matching `currencyId` against the API currency list. The platform's own admin has a full `/currencies` CRUD screen, so this map will silently break for any currency an operator adds.

The chat rules (`chatRulesContent`) are a hardcoded six-paragraph English string in the bundle: no spamming/harassment/caps, no begging for loans or rains or tips, no scams, no advertising or trading, no URL shorteners, use the right language room. Compare the player site, which carries `chatRule1`–`chatRule9` as separate translatable keys — a *different*, more operator-specific rule set including "Do not insinuate ARC has bad intent". The two rule sets are not in sync.

---

## 9. Security and Configuration Observations

These are incidental findings from reading a published bundle. No authorization testing, no fuzzing, no authenticated requests — the same standard as the rest of this investigation.

**8.1 — Everything runs over plaintext HTTP to a bare IP.** `http://45.198.14.55:7032` is both `VITE_APP_BASE_URL_1` and `SOCKET_URL`. Every REST call and the entire socket connection carry the player's session token to a raw IP:port with no TLS. The page itself is served over HTTPS from Cloudflare, so browsers will block these as mixed content in the first place — which is very likely a second contributor to the connection failures observed. But the deeper problem is that the intended design is unencrypted: a session token and (see below) an account password in transit, in the clear, bypassing the CDN, WAF and TLS termination that front everything else. This is the same host previously seen from the player site (`RETAIL-SOURCE-ANALYSIS.md` §7.1) and it is now confirmed as the chat's *entire* backend, not just a socket quirk.

**8.2 — The account password is sent base64-encoded on every tip.** `TipPlayer.vue` requires the player's login password to authorise a peer-to-peer transfer and sends it as `Buffer.from(password).toString("base64")`. Base64 is an encoding, not encryption; it is reversed by anyone who sees the request. Combined with 8.1, a player's **account password travels in trivially reversible form over unencrypted HTTP** whenever they tip. Requiring re-authentication for a value transfer is a reasonable control; this implementation defeats it. It should be TLS plus a server-side verify of the plaintext password against the stored hash, or better, a scoped confirmation token issued by the auth service. *(Observed in source. Not exercised — no tip was sent.)*

**8.3 — Origin allowlist baked in at build time, and wrong for this tenant.** §4.3. Three bare-IP HTTP origins, none of which is the actual parent. Beyond the immediate breakage, it forces a per-tenant rebuild of the chat app, or a shared allowlist that lets tenants reach into each other's chat frames.

**8.4 — Session token in `localStorage` on an origin with no CSP.** §4.4. `localStorage["accessToken"]`, persisted indefinitely, readable by any script on the chat origin. `curl -I` on `https://white-label-chat.gammaplus.io/` returns **no `Content-Security-Policy`, no `Strict-Transport-Security`, no `X-Frame-Options`, no `X-Content-Type-Options`, no `Referrer-Policy`** — and `Access-Control-Allow-Origin: *` on the HTML document. Same absence as the player site (`RETAIL-SOURCE-ANALYSIS.md` §7.2), but it matters more here because here the token is JS-readable.

**8.5 — Messages are rendered with `innerHTML` and the client does not escape them.** `ChatMessage.vue`:

```js
innerHTML: renderMentions(message.message, formatTime(message.createdAt, "humanized_time"), true)
```

where

```js
const renderMentions = (text, timeStr, asHtml) => {
  if (!text) return;
  const re = /@{([^:]+):[^}]+}/g;
  const out = text.replace(re, asHtml
    ? '<span class="text-highlightColor link-user" style="cursor: pointer;">@$1</span>'
    : "@$1");
  return asHtml
    ? `${out.replace(/\n/g, "<br>")}<div style="…">${timeStr}</div>`
    : out;
};
```

The raw message body is interpolated into an HTML string with no escaping and no sanitizer (no DOMPurify or equivalent is bundled — searched). `ReplyMessage.vue` does the same. Whether this is exploitable depends entirely on server-side sanitisation of `send-message`, which cannot be tested from a static read and was not attempted. It is worth noting that stored XSS in a chat panel that is iframed into every page of the casino, on an origin that holds the session token in `localStorage`, would be about the worst-placed XSS on the platform. The defence-in-depth fix is cheap: render with `v-text`/interpolation and build the mention spans as real VNodes rather than a string.

**8.6 — Sockets exposed on `window`.** `window.privateSockets = registry` in the chat app, `window.privateSocket = socket` in the player app. A live authenticated socket handle reachable from any script on the page. Debug aid left in a production build.

**8.7 — Hardcoded Giphy API key.** `new GiphyFetch("sXpGFDGZs0Dv1mmNFvYaGUvYwKX0PWIh")`, in the clear. Giphy web keys are necessarily client-side and this string is a widely circulated public/beta key rather than something operator-specific, so the exposure is low — but it is a third-party key committed to source, it is shared with whoever else uses that key, and it is subject to their rate limits.

**8.8 — Third-party CDN dependency at runtime.** Emoji images load from `https://cdn.jsdelivr.net/npm/emoji-datasource-apple@6.0.1/img/apple/64`, and GIFs from Giphy. Both are unpinned external origins inside a regulated gambling product; both would need to appear in any future CSP; both are availability dependencies nobody on the operator side controls.

**8.9 — Payload weight.** `event_chat_bg-BYODp8Dr.png` is **5.1 MB** for a chat background, in an app whose total download is 8.1 MB. The CSS bundle is 721 KB — larger than the entire lazy-loaded chat view — because Vuetify appears to ship unpurged. `moment.js` (deprecated, ~290 KB with locales) is bundled into the route chunk. None of this is a security issue; it is a straightforward cost on mobile players, and the chat panel loads on **every page** of the casino.

**8.10 — Two credential conventions for one token.** REST sends `Authorization: AccessToken=<t>`; the socket sends `access-token: <t>`. Both are custom. Neither is `Bearer`. Any gateway, WAF rule or log scrubber has to know about both.

---

## 10. What This Tells Us About the Platform

- **Chat is a genuinely separate product**, built by a different team or at a different time, on a different stack, against a different backend service, with its own auth convention and its own release cadence. The version skew in the postMessage enum (§4.2) is direct evidence that the two are not released together.
- **The iframe-plus-postMessage integration is the platform's answer to embedding a third-party-shaped component in the player site**, and it is the seam where the platform's otherwise-decent token handling degrades (§4.4). Anyone building this would want either a same-origin path mount (reverse-proxied under the player domain, so the HttpOnly cookie just works) or a short-lived scoped chat token minted server-side — not the long-lived session token in cross-origin `localStorage`.
- **Chat Rain, tipping and the shared big-win/achievement feed are real retention features**, not decoration, and each one couples chat to the wallet, the bonus engine and the tournament system. On a build-vs-buy assessment this is a component that looks small and is not: a from-scratch equivalent needs presence, history, moderation, per-channel eligibility rules, a scheduler, and value transfer between player wallets with the correctness requirements that implies.
- **The tenancy story is weaker here than anywhere else in the platform.** Tenant id is the literal `1` (§4.5), the trusted-origin list is a build-time constant (§4.3), and localisation is two hardcoded locales with no operator tooling (§8) against the player site's eight and the admin's translation editor. Whatever multi-tenant discipline the rest of the platform has, this app was not held to it.

---

## Appendix: File Inventory

```
chat/
├── index.html                              (430 B — SPA shell, <title>GS Chat</title>)
├── favicon.svg
├── assets/
│   ├── index-D2quYBNB.js                   (859 KB — app, stores, API layer, socket layer, i18n, most components)
│   ├── index-DRQF-Gex.css                  (721 KB)
│   ├── Channel-CiZQLSUV.js                 (510 KB — lazy: Channel.vue, chatbody, chatrain, moment.js)
│   ├── NotFoundPage-C8ySTa47.js            (519 B — lazy: 404)
│   ├── materialdesignicons-webfont-*.woff2 (403 KB)
│   ├── messageReceive-CtOLJ-sq.mp3         (57 KB — notification sound)
│   ├── event_image_fallback-*.webp         (12 KB)
│   └── {event_chat_bg (5.1 MB), chat-rain-bg (615 KB), bet_win_share_bg,
│        money_share_bg, tournament-bg}.png
└── images/tier1..5.svg                     (five loyalty-tier badges)
```

Not downloaded: `.eot` / `.ttf` / `.woff` variants of the icon font (superseded by the `.woff2`).
