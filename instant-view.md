# Instant View when blocked by an old template

Problem: an old approved IV template for your domain (possibly registered by
someone unrelated to you) keeps overriding Medium's fallback. It stays
opaque, so a site redesign silently breaks Instant View and there's no way
to inspect what the legacy rules expect.

Idea: serve Telegram Instant View (IV) pages from a separate, branded short
domain, without touching application code and without duplicating the backend.

## Background

Telegram Instant View is great for readers but painful to maintain. An IV
template is a private fragile rule set evaluated against your HTML; small markup
changes can silently break it, and getting a new template approved is slow.

A well-known workaround documented at
[nikstar.me/post/instant-view](https://nikstar.me/post/instant-view/) is to
make your pages look enough like Medium's that Telegram falls back to its
**already-approved Medium template**. No approval needed, no template of
your own to maintain.

There's a catch though. If anyone has ever submitted an approved IV template
for your domain in the past, including someone unrelated to you, Telegram
keeps using **that** template in preference to Medium's, and you can't see
what's inside it. This bites hard after a site redesign: the old rules no
longer match the new markup, Instant View looks broken, and you have no way
to inspect what the legacy template expects. You're left guessing.

The pattern below sidesteps this by routing Telegram to a fresh sibling
domain that has no IV template history. The Medium fallback then applies
cleanly to pages served from there.

## The trick

Two hostnames in front of one backend:

- **main.example**: the normal site.
- **short.example**: IV-only. Serves the **same page paths** as main
  (e.g. `/pages/...`, `/en/pages/...`); anything else gets bounced or blocked.

Decision-making is done by **edge rules** (CDN / redirect engine / WAF).
The app gets a request for a page URL and renders the IV variant when it
sees the short hostname. That's it.

The same effect can be reproduced in other ways: User-Agent sniffing in the
application itself, a small proxy in front of the origin, or a reverse-proxy
config that hands off based on host and UA. The setup described below is a
production-ready variant that keeps the logic at the edge (no extra moving
parts to operate) and out of the application code. The app team only needs
to add the Medium-fallback meta tags described at
[nikstar.me/post/instant-view](https://nikstar.me/post/instant-view/).

## Flow

```
1. Crawler visits the original URL on main
   ──────────────────────────────────────────
   TelegramBot ─▶ main.example/pages/foo
                  │
                  └─ rule: UA = TelegramBot AND host = main
                     302 ─▶ short.example/pages/foo

2. Crawler follows the redirect and fetches IV
   ─────────────────────────────────────────────
   TelegramBot ─▶ short.example/pages/foo
                  │
                  └─ no rule matches ─▶ backend ─▶ IV-flavored HTML
                                          (routes by Host header)

3. A human clicks the IV link in Telegram, opens in a browser
   ────────────────────────────────────────────────────────────
   Chrome ─▶ short.example/pages/foo
            │
            └─ rule: UA ≠ TelegramBot
               302 ─▶ main.example/pages/foo

4. Stray traffic on the short host
   ─────────────────────────────────
   anything not under /pages/ or /en/pages/ ──▶ 302 to main.example
                                                  (or blocked at the WAF)
```

## Building blocks

- One redirect rule on the main zone (UA-gated, host-scoped).
- A pair of redirect rules on the short zone: a path-prefix allow-list bounce
  and a UA bounce; both send mismatched requests to main with the URI
  preserved.
- A WAF allow-list on the short zone for the same page path prefixes
  (defense in depth; the redirect rules normally short-circuit traffic
  before the WAF gets a turn, so this is the safety net for when redirects
  are disabled or misconfigured).
- DNS on both zones pointing at the same backend; the backend routes by
  hostname and treats requests on the short host as the IV variant.

That's the whole pattern. The interesting bit is that the application doesn't
have to know anything about Telegram: the UA check happens at the edge, and
the backend just decides "IV or normal?" based on which hostname it was
asked for.
