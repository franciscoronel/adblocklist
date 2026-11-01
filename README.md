# Ad, ad-mediation and offer-wall hosts

Ad, ad-mediation and offer-wall domains that shipping Android apps actually reference — for apps
whose ads are fronted by **TradPlus** and mediated to **AppLovin MAX**, **Pangle (TikTok)**,
**Unity Ads**, **Vungle** and **SafeDK**. In [StevenBlack/hosts](https://github.com/StevenBlack/hosts)
format, so it drops straight into Pi-hole, AdGuard Home, Unbound or a hosts file.

**Subscribe:** `https://raw.githubusercontent.com/franciscoronel/adblocklist/main/hosts`

**Unique domains: 105** · updated 1 November 2026 14:58:43 (UTC)

## What is in it

Every entry is a host literal taken from the string/asset tables of a store-signed Android app's
APK (apktool-decoded) and cross-checked against DNS traffic captured on the device. Nothing here
was guessed from "typical ad SDK domains", and nothing app-specific is listed — this file names
ad-serving infrastructure only.

| family | entries |
|---|---|
| TRADPLUS MEDIATION  (block these first) | 29 |
| GOOGLE AD MOB | 6 |
| APPLOVIN (MAX) | 32 |
| PANGLE (TikTok) | 9 |
| UNITY ADS | 11 |
| VUNGLE | 7 |
| SAFEDK OFFER-WALL  (source of the sponsored auto-installs) | 2 |
| Declared only in the mediation waterfall config (no live sub-domain literal in the APK) | 9 |

Sections that matter most:

- **TradPlus** — the mediation header. Block it first: when the header has no config to return,
  the waterfall never fans out to the mediated networks, so AppLovin / Pangle / Unity / Vungle
  traffic stops without a rule per network. Its endpoints are on none of the mainstream
  blocklists, which is why apps keep filling ads even with those networks already filtered.
- **SafeDK** — `edge.safedk.com` / `config.safedk.com` are the offer-wall SDK behind "sponsored
  app" auto-instals, and they show up in *live device traffic*, not only in code.
- **Declared-only** — apex domains that the mediation config references (InMobi, IronSource,
  Chartboost, Moloco, bidmachine.io, tradplusad.com). No live sub-domain literal was seen for
  them; blocking them costs nothing and covers the fallback path.
- **Telemetry** — commented out so you can decide. Blocking them is safe for app behaviour but
  silences the host app's own Firebase/Crashlytics reporting.

## Import (Pi-hole)

`Settings → Blocklists → Add blocklist` → URL = the raw link above, list type **Hosts file**.

For whole families, a regex/wildcard rule is more durable than any hosts file, because these SDKs
add sub-domains and answer geo-anchored (the same name returns different IPs to different
resolvers):

```
regex:tradplusad\.com$
regex:(applovin|applvn)\.com$
regex:(pangle\.io|pangleglobal\.com|tiktokpangle[a-z-]*\.(us|com)|tiktokv\.us)$
regex:vungle\.com$
regex:(unityads\.unity3d\.com|unity3dusercontent\.com)$
```

Verification method worth copying: query a name that cannot exist (`probe599480260.tradplusad.com`).
A filtering resolver answers `0.0.0.0`, a plain resolver answers NXDOMAIN — that contrast is what
proves a regex covers the whole family, including sub-domains added later. Run the host app's own
API/CDN hosts in the same batch to prove there is no false match.

## Two things a blocklist cannot do

1. **The device must actually use the filter.** A rule on a resolver changes nothing for a device
   that points somewhere else: if DHCP hands out an unfiltered resolver and Private DNS is off,
   every rule here is bypassed. Check the path the device really uses first:
   `dig @<resolver-the-device-actually-uses> a.applovin.com`.
2. **IPs are fragile, domains are not.** Geo-anchored answers rotate per resolver, and ad-network
   infrastructure shares cloud address space with app infrastructure. That is why this file has no
   CIDR ranges — an address rule would block the app along with the ads.

## What is *not* blocked

Hosts belonging to an app's own API, config, static assets, video CDN, load balancers or player
licence are deliberately excluded, and no such hostnames are named anywhere in this repo. Adding
them would break playback, not ads; the `DO NOT BLOCK` note in `hosts` states the rule so future
edits do not drift into app infrastructure.

## Provenance / rebuild

Source: one store-signed Android app (versionCode 2061001, signing cert `CN=Android,O=Google Inc.`),
installed from the store, split-APK set.
Method: `apktool` decode → host-literal extraction over `smali*/`, `res/`, `assets/` → ad-family
filter → drop app/infrastructure hosts → cross-check against captured device DNS.
Re-run per app version: which networks an app mediates to changes between releases.

Unverified by measurement, for any app using this stack: whether the app still serves *cached* or
app-owned offer-wall content once the mediation header is unreachable. That is a device-side test,
not a list problem.

## License

MIT
