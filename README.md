# Cache Buster 9000

> **[→ View the full interactive version at wendy.monster/cache-buster-9000/](https://wendy.monster/cache-buster-9000/)** — cleaner to read there (tabbed server configs, copy buttons, coverage matrix).

Every known technique to prevent browsers from serving stale pages. Server headers, HTML meta, JS runtime traps — all of them, in one reference. Includes a drop-in client template.

---

**No single technique covers everything.** iOS Safari's Back-Forward Cache ignores all HTTP headers. Some CDNs ignore `Expires`. Android Chrome caches `fetch()` responses separately from page loads. Service Workers can override everything. This reference combines every known layer so nothing slips through.

---

## Server-Side Headers

### S1 — Anti-Cache Response Headers
*all browsers*

Set these headers on every HTML response. Together, they cover HTTP/1.0 proxies, HTTP/1.1 caches, shared CDN caches, and all modern browsers. This is the single most important layer.

**nginx**
```nginx
# Add to your server or location block
add_header Cache-Control "no-store, no-cache, max-age=0, must-revalidate, proxy-revalidate" always;
add_header Pragma        "no-cache" always;
add_header Expires       "0" always;
add_header Vary          "*" always;
add_header Surrogate-Control "no-store" always;

# "always" ensures headers are set on ALL response codes, not just 2xx/3xx
```

**Apache**
```apache
# Requires mod_headers (usually enabled by default)
# Add to .htaccess, <Directory>, or <Location> block
Header always set Cache-Control "no-store, no-cache, max-age=0, must-revalidate, proxy-revalidate"
Header always set Pragma        "no-cache"
Header always set Expires       "0"
Header always set Vary          "*"
Header always set Surrogate-Control "no-store"

# "always" ensures headers are set even on error responses (4xx, 5xx)
```

**Caddy**
```
# Add to your site block in Caddyfile
header {
    Cache-Control      "no-store, no-cache, max-age=0, must-revalidate, proxy-revalidate"
    Pragma             "no-cache"
    Expires            "0"
    Vary               "*"
    Surrogate-Control  "no-store"
}
```

**lighttpd**
```
# Requires mod_setenv
server.modules += ( "mod_setenv" )

setenv.add-response-header = (
    "Cache-Control"     => "no-store, no-cache, max-age=0, must-revalidate, proxy-revalidate",
    "Pragma"            => "no-cache",
    "Expires"           => "0",
    "Vary"              => "*",
    "Surrogate-Control" => "no-store"
)
```

**Node.js (Express)**
```js
// Express / Connect middleware
app.use(function(req, res, next) {
  res.set('Cache-Control', 'no-store, no-cache, max-age=0, must-revalidate, proxy-revalidate');
  res.set('Pragma',         'no-cache');
  res.set('Expires',        '0');
  res.set('Vary',           '*');
  res.set('Surrogate-Control', 'no-store');
  next();
});
```

| Header | What it does |
|--------|-------------|
| `Cache-Control: no-store` | Strongest directive ([RFC 7234 §5.2.2.3](https://www.rfc-editor.org/rfc/rfc7234#section-5.2.2.3)). Tells every cache — browser, proxy, CDN — to never store any part of the response. `no-cache` forces revalidation on every use. `must-revalidate` prevents serving stale content even when disconnected. `proxy-revalidate` extends this to shared caches. |
| `Pragma: no-cache` | HTTP/1.0 backward compatibility. Some older proxies and corporate network appliances only understand HTTP/1.0 headers. Harmless to include. |
| `Expires: 0` | Marks the response as already expired. Technically the spec expects a date string, but `0` is universally treated as "expired" by all known implementations. |
| `Vary: *` | Signals the response varies on factors beyond request headers ([RFC 7231 §7.1.4](https://www.rfc-editor.org/rfc/rfc7231#section-7.1.4)). Shared caches (CDNs, reverse proxies) will never reuse this response for another request. |
| `Surrogate-Control: no-store` | CDN-specific header recognized by Fastly, Varnish, Akamai, and other edge caches. Controls CDN caching independently from `Cache-Control`. Stripped before reaching the browser. |

---

### S2 — Build Version Injection
*all browsers · server required*

The server replaces a placeholder string (`YOUR_BUILD_HASH`) in the HTML with a unique value on each deploy — a content hash, git commit, or timestamp. The client JS (S3 and S4 below) then uses this value to detect stale pages. **Without this step, S3 and S4 are inert — they won't hurt anything, but they won't fire either.** Most build tools (Webpack, Vite, etc.) can inject this at build time instead of at serve time.

**nginx**
```nginx
# Requires ngx_http_sub_module (included in most nginx builds)
# Add to your server or location block
set $build_hash "abc123";  # Update this value on every deploy
sub_filter 'YOUR_BUILD_HASH' $build_hash;
sub_filter_once on;
sub_filter_types text/html;

# NOTE: sub_filter does not work on gzip-compressed responses.
# If you serve gzipped HTML, disable gzip for this location:
# gzip off;
# Or if proxying: proxy_set_header Accept-Encoding "";
```

**Apache**
```apache
# mod_substitute does NOT support env var expansion — hardcode the hash directly.
# Requires mod_substitute. Add to .htaccess or <Location> block.
# Update the hash value on each deploy (via deploy script or CI).
AddOutputFilterByType SUBSTITUTE text/html
Substitute "s/YOUR_BUILD_HASH/abc123/n"

# Alternatively, use a deploy-time sed command (same as lighttpd approach):
# sed -i "s/YOUR_BUILD_HASH/$BUILD_HASH/g" /path/to/index.html
```

**Caddy**
```
# Use Caddy's templates middleware to inject a variable
# In your Caddyfile site block:
templates

# Set an environment variable before starting Caddy:
# BUILD_HASH=abc123 caddy run
# Then in your HTML, replace YOUR_BUILD_HASH with:
# {{env "BUILD_HASH"}}
```

**lighttpd**
```bash
# lighttpd has no built-in HTML substitution.
# Recommended: inject the hash at build time with a sed command in your deploy script:
#
# BUILD_HASH=$(git rev-parse --short HEAD)
# sed -i "s/YOUR_BUILD_HASH/$BUILD_HASH/g" /path/to/index.html
#
# Or use any build tool (Vite, Webpack, Parcel) to inject at build time.
```

**Node.js**
```js
// Read and inject the hash when serving the HTML file
const fs = require('fs');
const BUILD_HASH = process.env.BUILD_HASH || require('./package.json').version;
const html = fs.readFileSync('./public/index.html', 'utf8')
  .replace('YOUR_BUILD_HASH', BUILD_HASH);

app.get('/', (req, res) => {
  res.set('Content-Type', 'text/html');
  res.send(html);
});
```

---

### S3 — URL Version Hash
*all browsers · requires S2*

Appends the build hash as a `?v=` query parameter. Each deploy produces a different URL — the browser has no cache entry for it and must fetch fresh. This is the most reliable technique because it forces a new request even when every other mechanism fails. **Server-side setup: S2 only** — once the build hash is injected, this is purely client JS with no additional server configuration. If the placeholder hasn't been replaced, this block exits immediately — no redirect loops, no broken behavior.

```js
// [S3] URL versioning — redirect to ?v=BUILD if stale or missing
(function() {
  var BUILD = 'YOUR_BUILD_HASH';
  // Guard: split string so server injection only replaces the variable above,
  // not the sentinel. If BUILD still equals the placeholder, injection isn't set up.
  if (BUILD === 'YOUR_' + 'BUILD_HASH') return;
  var p = new URLSearchParams(window.location.search);
  if (p.get('v') !== BUILD) {
    p.set('v', BUILD);
    window.location.replace(window.location.pathname + '?' + p.toString());
  }
})();
```

---

### S4 — localStorage Version Sentinel + Tab Watcher
*all browsers · requires S2*

Stores the build version in `localStorage`. On load, if the stored version differs from the current one, the page is stale — reload. A `visibilitychange` listener extends this to long-lived tabs: when the user switches back after a deploy, the check runs again. **Server-side setup: S2 only** — all server config is covered by S2, this is purely client JS. If the placeholder hasn't been replaced, both checks are skipped entirely — safe to include unconditionally.

```js
// [S4] localStorage sentinel + visibilitychange
(function() {
  var BUILD = 'YOUR_BUILD_HASH';
  // Split string so server injection only replaces the variable, not the sentinel.
  if (BUILD === 'YOUR_' + 'BUILD_HASH') return;
  var KEY = 'app_build';
  try {
    var stored = localStorage.getItem(KEY);
    if (stored && stored !== BUILD) {
      localStorage.setItem(KEY, BUILD);
      window.location.reload();
      return;
    }
    localStorage.setItem(KEY, BUILD);
  } catch(e) {} // localStorage blocked in some private browsing modes
  document.addEventListener('visibilitychange', function() {
    if (document.visibilityState === 'visible') {
      try {
        if (localStorage.getItem(KEY) !== BUILD) window.location.reload();
      } catch(e) {}
    }
  });
})();
```

---

## Client-Side Techniques

### C1 — HTML Meta Tags
*limited effect*

Duplicate cache directives as HTML `<meta>` tags. **Important caveat:** modern browsers largely ignore `http-equiv` cache directives in favor of actual HTTP headers. The HTML Living Standard does not list `Cache-Control` as a valid `http-equiv` value. However, some Android WebViews, embedded browsers, RSS readers, and older crawlers still parse them. They cost nothing to include and occasionally help.

```html
<!-- Place in <head>, before any other tags -->
<meta http-equiv="Cache-Control" content="no-store, no-cache, max-age=0, must-revalidate">
<meta http-equiv="Pragma" content="no-cache">
<meta http-equiv="Expires" content="0">
```

---

### C2 — Back-Forward Cache Bypass
*iOS Safari · all browsers*

When you hit the back button, browsers can restore a frozen in-memory snapshot of the page (the **bfcache**) instead of making a network request. This completely bypasses all HTTP headers. iOS Safari uses this aggressively. The `pageshow` event fires with `event.persisted === true` when restoring from bfcache. Force a real network reload in that case. *Supported in all major browsers since 2015+, iOS Safari since iOS 5.*

```js
// Place early in your script, before app initialization
window.addEventListener('pageshow', function(e) {
  if (e.persisted) {
    window.location.reload();
  }
});
```

---

### C3 — fetch() cache: 'no-store'
*all browsers*

The Fetch API maintains its own interaction with the browser's HTTP cache. Setting `cache: 'no-store'` tells the browser to skip the cache entirely — it won't use a cached response, and it won't store the new response. This prevents stale API responses even when the browser has cached previous calls to the same URL. *Supported in all current browsers. The `cache` request option specifically requires Chrome 64+, Firefox 61+, Safari 10.1+.*

```js
// Add to every fetch() call in your API wrapper
var response = await fetch(url, {
  method: method,
  headers: { 'Content-Type': 'application/json' },
  cache: 'no-store'
});
```

---

### C4 — Service Worker Unregistration
*all browsers*

A previously registered Service Worker can intercept all network requests via its `fetch` event handler and serve cached responses, completely bypassing every other cache-busting technique. Defensively unregister any existing Service Workers on every page load. **Note:** after `unregister()` is called, the SW remains active until all its controlled pages close — but it won't intercept new navigations.

```js
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.getRegistrations().then(function(regs) {
    regs.forEach(function(r) { r.unregister(); });
  });
}
```

---

## Drop-in Client Template

All client-side layers — paste into your HTML.

> **This template covers client-side layers only.** Server-side response headers (Cache-Control, Pragma, Expires, Vary, Surrogate-Control) must be configured separately in your web server — see the *Server-Side Headers* section above. Without those headers, browsers and proxies may still cache your pages regardless of what the client-side JavaScript does. The client techniques are a safety net, not a replacement for proper HTTP headers.

> **The S3 and S4 blocks (URL versioning and localStorage sentinel) require server-side build hash injection to have any effect.** If `YOUR_BUILD_HASH` is never replaced, those blocks exit immediately — they are not harmful and will not cause redirect loops or errors, but they simply won't fire. To activate them, configure your server or build tool to substitute the placeholder with a unique value on each deploy (see *S2: Build Version Injection* above).

```html
<!-- Cache Buster 9000: Client-Side Layers -->
<!-- Place these meta tags at the top of <head> -->
<meta http-equiv="Cache-Control" content="no-store, no-cache, max-age=0, must-revalidate">
<meta http-equiv="Pragma" content="no-cache">
<meta http-equiv="Expires" content="0">

<script>
// === CACHE BUSTER 9000 ===
// Place this block as the FIRST thing inside your <script> tag,
// before any other application code.

// [C2] bfcache bypass — iOS Safari back button
window.addEventListener('pageshow', function(e) {
  if (e.persisted) window.location.reload();
});

// [S3 + S4] URL versioning + localStorage sentinel + visibilitychange
// Requires server-side build hash injection (see S2 in the Server-Side section).
// If YOUR_BUILD_HASH hasn't been replaced by your server or build tool,
// these blocks exit immediately — safe to include unconditionally.
(function() {
  var BUILD = 'YOUR_BUILD_HASH';
  var KEY = 'app_build';

  // [S3] URL versioning: redirect to ?v=BUILD if query param is stale
  // Guard uses split string so server injection only replaces the variable above.
  if (BUILD !== 'YOUR_' + 'BUILD_HASH') {
    var p = new URLSearchParams(window.location.search);
    if (p.get('v') !== BUILD) {
      p.set('v', BUILD);
      window.location.replace(window.location.pathname + '?' + p.toString());
      return; // stop executing — page is about to redirect
    }
  }

  // [S4] localStorage sentinel: detect stale cached page
  if (BUILD !== 'YOUR_' + 'BUILD_HASH') {
    try {
      var stored = localStorage.getItem(KEY);
      if (stored && stored !== BUILD) {
        localStorage.setItem(KEY, BUILD);
        window.location.reload();
        return;
      }
      localStorage.setItem(KEY, BUILD);
    } catch(e) {}

    // visibilitychange: catch long-lived stale tabs
    document.addEventListener('visibilitychange', function() {
      if (document.visibilityState === 'visible') {
        try {
          if (localStorage.getItem(KEY) !== BUILD) window.location.reload();
        } catch(e) {}
      }
    });
  }
})();

// [C4] Service Worker unregistration
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.getRegistrations().then(function(regs) {
    regs.forEach(function(r) { r.unregister(); });
  });
}

// [C3] For your fetch() wrapper, add: cache: 'no-store'
// Example: fetch(url, { method: 'GET', cache: 'no-store' })
</script>
```

---

## Coverage Matrix

| Scenario | Technique |
|----------|-----------|
| ✓ Browser HTTP cache | Cache-Control: no-store (S1) |
| ✓ CDN & reverse proxy caching | Vary: * + Surrogate-Control (S1) |
| ✓ Legacy HTTP/1.0 proxies | Pragma + Expires (S1) |
| ✓ Stale cached page load | URL versioning (S3) + localStorage sentinel (S4) |
| ✓ Long-lived tabs after deploy | visibilitychange (S4) |
| ✓ iOS Safari back button | bfcache bypass (C2) |
| ✓ Cached API / fetch() responses | cache: 'no-store' (C3) |
| ✓ Rogue Service Workers | SW unregister (C4) |
| ✓ WebViews & embedded browsers | HTML meta tags (C1) |

---

*Cache Buster 9000 — assembled by Wendy after one too many "why is iOS showing old stuff" incidents*
