# Ad, ad-mediation and offer-wall hosts

Ad, ad-mediation and offer-wall domains that shipping Android apps actually reference — from two
short-drama apps audited on a real device:

| app | package | monetisation stack |
|---|---|---|
| DramaWave | `com.dramawave.app` | **TradPlus** header → AppLovin MAX, Pangle, Unity, Vungle, SafeDK |
| Melolo (Worldance) | `com.worldance.drama` | **AppLovin MAX** header with Mintegral / Facebook / Google / InMobi / Moloco adapters, **Pangle PAG** (TTMediation SDK 6.5.1.1, `com.bytedance.sdk.pangle.version` 7.8.8.3) waterfall (adapters: InMobi, Chartboost, Mintegral, IronSource, BidMachine), Unity, Vungle, AdMob, SafeDK 5.12.9 |

Both apps sit on the **same monetisation SDK family** (same SafeDK version, same adapter set, the
same `dragon.read` / `tmtreader` publisher stack underneath), so the families overlap — but the
**header differs**, and that is the operational difference that matters (see below).

In [StevenBlack/hosts](https://github.com/StevenBlack/hosts) format, so it drops straight into
Pi-hole, AdGuard Home, Unbound or a hosts file.

**Subscribe:** `https://raw.githubusercontent.com/franciscoronel/adblocklist/main/hosts`

**Unique domains: 139** · updated 2 November 2026 (UTC)

## What is in it

Every entry is a host literal taken from the string/asset tables of a store-signed Android app's
APK (decoded manifest + `dex`/`assets`/`res` scan) and cross-checked against DNS traffic captured
on the device. Nothing here was guessed from "typical ad SDK domains", and nothing app-specific is
listed — this file names ad-serving infrastructure only. Hosts that are *declared* but not *named*
(the adapter bundles in AppLovin MAX's waterfall config) are covered by the network's live
endpoints, found by DNS, plus an apex entry in the declared-only section.

| family | entries | note |
|---|---|---|
| TRADPLUS MEDIATION (block these first) | 29 | DramaWave's header |
| GOOGLE AD MOB | 7 | incl. `ad.doubleclick.net` |
| APPLOVIN (MAX) | 32 | Melolo's header |
| APPLOVIN MAX MEDIATION ADAPTERS | 3 | `an.facebook.com`, `api2.facebook.com`, `api.mintegral.com` |
| PANGLE (TikTok) | 21 | ad serving + mediation + the SDK's own `mon.`/`log.` ad-app monitoring |
| INMOBI | 8 | adapter under *both* headers (live SDK sub-domains) |
| UNITY ADS | 13 | incl. `config.unityads…`, `gateway-b.offerwall…`, `fill.adserverapp.com` |
| VUNGLE | 7 | |
| MOLOCO | 1 | `cdn-f.adsmoloco.com` (API hosts fall under the `moloco.com` apex) |
| MINTIGRAL / MTG | 2 | `api.mintegral.com`, `api.ssdk.cn`, `mtgglobals.com` |
| ATTRIBUTION / AD-SYNC | 3 | `aps-webhandler.appsflyer.com`, `spadsync.com`, `www.googleadservices.com` |
| TAPJOY offer-wall | 1 | `api.tapjoy.com` |
| SAFEDK OFFER-WALL (source of the sponsored auto-installs) | 2 | |
| Declared only in the mediation waterfall config | 9 | apex fallbacks (InMobi, IronSource, Chartboost, Moloco, bidmachine.io, tradplusad.com …) |

Sections that matter most:

- **The header is the lever, per app.** TradPlus (DramaWave) and AppLovin MAX (Melolo) are
  different headers: blocking TradPlus does nothing for Melolo, and vice-versa. Block the header a
  given app actually uses first — when the header has no config to return, the waterfall never fans
  out to the mediated networks. TradPlus endpoints are on none of the mainstream blocklists, which
  is why those apps keep filling ads even with AppLovin/Pangle/Unity/Vungle already filtered.
- **Adapters carry no host literals.** AppLovin MAX's waterfall names Mintegral, Facebook Audience
  Network, Google, InMobi and Moloco inside `META-INF/com/applovin/mediation/*-adapter` but ships
  no hostname for them; the traffic appears on each network's own hosts. That is why this file
  includes the adapter networks' live endpoints, not just the header's.
- **SafeDK** — `edge.safedk.com` / `config.safedk.com` are the offer-wall SDK behind "sponsored
  app" auto-instals, and they show up in *live device traffic*, not only in code (both apps ship
  SafeDK 5.12.9).
- **Declared-only** — apex domains the mediation config references. No live sub-domain literal was
  seen for them; blocking them costs nothing and covers the fallback path.
- **Telemetry** — deliberately *not* in `hosts` (that file stays comment-lean: one section header
  per family, entries only). These are safe to block for app behaviour but silence the host app's
  own Firebase/Crashlytics and analytics reporting, so they are listed here as opt-ins to paste into
  your own resolver rules if you want them: `app-measurement.com`, `www.google-analytics.com`,
  `fundingchoicesmessages.google.com`, `csi.gstatic.com`, `assets.adobedtm.com`,
  `firebase-settings.crashlytics.com`, `firebaseinstallations.googleapis.com`,
  `firebaseremoteconfig.googleapis.com`, `firebaseremoteconfigrealtime.googleapis.com`.

## Facebook Audience Network: deliberate partial coverage

The AN SDK ships inside Melolo (`assets/audience_network.dex`,
`com.facebook.ads.AudienceNetworkContentProvider`) and its ad hosts resolve (`an.facebook.com`,
`api2.facebook.com` → same edge IP). But they live under `facebook.com`, which the app also uses
for **login/OAuth and share**, so only the AN-specific names are blocked and bare
`facebook.com` / `www.facebook.com` / `connect.facebook.net` are left reachable. The AN event log
(`www.facebook.com/adnw_logging`) needs a URL/path rule, not a domain rule.

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
regex:(isnssdk|sgsnssdk)\.com$
regex:mtgglobals\.com$
regex:inmobi(cdn)?\.com$
```

Verification method worth copying: query a name that cannot exist (`probe599480260.tradplusad.com`).
A filtering resolver answers `0.0.0.0`, a plain resolver answers NXDOMAIN — that contrast is what
proves a regex covers the whole family, including sub-domains added later. Run the host app's own
API/CDN hosts in the same batch to prove there is no false match.

## Three things a blocklist cannot do

1. **The device must actually use the filter.** A rule on a resolver changes nothing for a device
   that points somewhere else: if DHCP hands out an unfiltered resolver and Private DNS is off,
   every rule here is bypassed. Check the path the device really uses first:
   `dig @<resolver-the-device-actually-uses> a.applovin.com`.
   Measured 2026-11-02 on this LAN: the filtering Pi-hole answers `0.0.0.0` for the DramaWave
   hosts (so the deployed list works), but a phone on the same Wi-Fi still resolved
   `a.applovin.com`, `a.pangleglobal.com` and `lf-static.tiktokpangle-cdn-us.com` to real ad IPs —
   i.e. it is **not** using the filtering resolver. Fix the path before crediting the list.
2. **An SDK can resolve names itself.** This app family ships its own networking stack
   (`com.bytedance.ttnet` + Cronet 107; the Pangle bridge logs `AVMDLManagerBridge init_dns_info`),
   so some ByteDance hosts may be reached from the SDK's own DNS data instead of the system
   resolver. Domain rules cover the AppLovin / Google / Facebook / Unity / Vungle / Moloco /
   Mintegral / InMobi side reliably; watch device query logs for the Pangle side.
3. **IPs are fragile, domains are not.** Geo-anchored answers rotate per resolver, and ad-network
   infrastructure shares cloud address space with app infrastructure. That is why this file has no
   CIDR ranges — an address rule would block the app along with the ads.

## App-side alternatives (when DNS filtering is not enough)

- **Device advertising ID** — neither app declares `AD_ID`/`RENDER_AD` in its manifest, so the lever
  is device-level: **Settings → Privacy → Ads → Delete advertising ID / Opt out of ads**. It removes
  personalisation, not the ad requests.
- **The header initialises from a `ContentProvider`** — that is the surgical disable point for a
  self-signed re-patch, and it is exactly where the ad SDKs switch themselves on:
  `com.applovin.sdk.AppLovinInitProvider` (MAX header), `com.google.android.gms.ads.MobileAdsInitProvider`
  (AdMob), `com.vungle.ads.VungleProvider`, `com.facebook.ads.AudienceNetworkContentProvider`.
  Google's AdMob app id is a manifest meta-data (`com.google.android.gms.ads.APPLICATION_ID =
  ca-app-pub-5861813288185762~9826446215`); AppLovin's SDK key is *not* a manifest meta-data here,
  so it is supplied from app config at runtime and cannot be blanked in the manifest.
- **Keep these providers** (they are not ads): `com.facebook.FacebookContentProvider` (share links
  target, e.g. Messenger), `com.bytedance.ug.sdk.share.impl.provider.ShareSdkProvider` (share SDK),
  `com.dragon.reader.lib.context.ReaderContentProvider` + `com.dragon.reader.simple.SimpleContentProvider`
  (the reader core), `com.bytedance.push.*` / `com.ss.android.push*` (notifications),
  `com.bytedance.bdinstall.BDInstallProvider` (SafeDK-adjacent install tracking), and
  `androidx.startup.InitializationProvider`.
- **Self-signed re-patch**: as with DramaWave, patching the APK (drop ad/tracking meta-data and
  disable the init providers above) works, but the rebuilt APK is signed with our own key —
  Play/GetApps can then never auto-update it or restore the ad SDKs, so every future app update
  needs a manual re-patch. DNS filtering avoids that maintenance cost, which is why the blocklist
  is the primary lever.

## What is *not* blocked

Hosts belonging to an app's own API, config, static assets, video CDN, load balancers or player
licence are deliberately excluded. For the two apps audited that specifically means:
`dramawave.com` family; and for Melolo/Worldance `worldance.drama`, `*.melolo.org`,
`*.tmtreader.com` (`api`, `api-my`, `passport`, `gab`, `log` — the app's own API and monitoring),
`fizzo.org`, `fictum.org`, `*.pipopay.com` + `pipopayment.us` (payment), `*.tiktokcdn*`,
`*.bytegecko*`, `*.ibytedtos.com`, `*.ibyteimg.com`, `byteintl.net`, `*.byteoversea.com`
(`vcs-*`, `verification-*`, `frontier` = login, push, A/B config), `*.snssdk.com` login endpoints,
`facebook.com` login/share, and Google APIs/Firebase used for auth, push and crash reporting.
The `DO NOT BLOCK` note in `hosts` states the rule so future edits do not drift into app
infrastructure; the app-owned names are listed here, in the README, so they are never mistaken for
a gap in the blocklist.

Note the overlap that a domain blocklist cannot resolve: the ByteDance *monitoring* hosts
(`mon.isnssdk.com`, `log.isnssdk.com`, `analytics.tiktok.com`, `mon.zijieapi.com`,
`gecko16-normal-useast5.tiktokv.us`) are Pangle-SDK ad/app monitoring, but the same SDK also
carries the app's own login/push/config traffic on sibling hosts. Only the ad/monitoring names are
listed, and the app-side note records exactly which ones to re-open if login or push misbehaves.

## Provenance / rebuild

Source: two store-signed Android apps, installed from the store, split-APK sets:
`com.dramawave.app` versionCode 2061001 (signing cert `CN=Android,O=Google Inc.`) and
`com.worldance.drama` 5.5.0 / versionCode 55018 (minSdk 23, targetSdk 35; base + 2 config splits,
~200 MB). Play-installed confirmed on-device (`installerPackageName=com.android.vending`); Melolo's
signing cert was **not** verified locally (no Java/`apksigner` on the testbed host — a v2 signing
block containing `CN=Android Debug` and an AppsFlyer jar cert was seen, which is not the signer
identity).
Method: pull the split set from the device over adb → decode manifest and resource tables →
host-literal extraction over the `dex`/`res`/`assets` payload plus the manifest metadata and
mediation config → ad-family filter → drop app/infrastructure hosts → **DNS existence check per
candidate** (a name with no A record is not a real endpoint) → live cold-start device logcat +
resolver comparison across the filtering resolver, the gateway resolver and a public resolver.
Deliberately *not* claimed: the runtime request set, the live waterfall order, and which ads are
actually served (no root, no proxy, and the app uses its own HTTP stack) — this list is a ceiling,
not a measured hit rate.
Re-run per app version: which networks an app mediates to changes between releases.

Unverified by measurement, for any app using this stack: whether the app still serves *cached* or
app-owned offer-wall content once the mediation header is unreachable. That is a device-side test,
not a list problem.

## License

MIT
