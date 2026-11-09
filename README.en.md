# Disabling the ColorOS Stock Browser (`com.heytap.browser`)

> ⚠️ **Risk notice: this investigation is not exhaustive, and disabling system components carries risk.** The conclusions come from static analysis of a single device plus limited hands-on testing; they do not cover every ColorOS version, model, or usage pattern. Disabling may cause: system entry points that fail to open, loss of built-in download / account / quick-app capabilities, and guest or multi-user spaces left without a browser. Evaluate first, back up, make sure you can roll back, and only then proceed.

A reproducible record of the **static analysis, risk assessment, disabling, and rollback** of the preinstalled browser shipped with ColorOS / realme.

1. What this package actually is (a full Chromium with 59,944 classes — not an ad SDK)
2. What you lose and what you gain by stopping it (checked item by item against calls, SMS, data, camera, payments, system updates)
3. How to stop it, verify it, and roll it back (including the difference between suspend and disable, and the failure caused by dual package records)

## Benefits

1. **Reclaims roughly 1 GB of resident memory** — resident from boot, not only while in use.
2. **Stops three things it never stops on its own** — feed prefetching (content downloaded ahead of your tap), ad-exposure attribution reporting (`oaid` plus attribution domains), and three keep-alive services with no permission protection (any app can bind them).
3. **Removes a ~59k-class privileged app from memory** — it holds `MANAGE_EXTERNAL_STORAGE`, `QUERY_ALL_PACKAGES`, `PACKAGE_USAGE_STATS`, and `REQUEST_INSTALL_PACKAGES`.
4. **Cuts one complete ad chain** — it is at once an ad surface (in-feed native ads), a landing-page container (other apps' ads open inside it), and a search-revenue entry point.
5. **Shrinks the attack surface** — `AdShowActivity` and `WebExportActivity` are both exported with no permission protection; the latter can load an arbitrary URL.

## Costs

1. **Every system entry point that targets it by name stops working** — these calls bypass the default browser and point directly at `com.heytap.browser` or its private schemes (`heytapbrowser://`, `opluspay://`, `ireaderplugin://`). They do not fall back to Chrome; they show "this app is managed by Shell". **How many such entry points exist is not enumerated here.**
2. **Guest mode has no usable browser** — the guest user (user 10) has this package but not Chrome; the multi-app user's (user 999) `BROWSER` role points at it, yet no browser is installed there.
3. **Some built-in capabilities disappear with it** — the downloader (bundled Thunder module), the quick-app bridge, bookmark and account sync, and some vendor in-app pages that render with the browser engine. The last item was **not verified case by case**.
4. **The data is still there, but unreachable** — bookmarks, history, and sessions remain under `/data/data/com.heytap.browser`; there is simply no UI left to read them. `unsuspend` brings them back as they were.
5. **This is not a permanent state** — `pm suspend` is per-user and stable day to day; a full OTA that rewrites `/product`, or a manual `unsuspend`, restores it.

## Known gaps in this investigation

This is an **incomplete** record. The following three items were not finished; test them yourself before copying anything here:

- No exhaustive list of system entry points that call `com.heytap.browser` by name or use its private schemes, so no exact number for "how many things will break".
- No verification of how vendor in-app pages rendered with the browser engine (agreement pages, campaign pages) behave after disabling.
- No hands-on test of opening links inside guest or multi-app spaces; only static checks of roles and package presence.

## Contents

- [Full write-up: analysis, trade-offs, and a reproducible procedure](docs/heytap-browser-lockdown.md) (Chinese)

## Scope and prerequisites

| Item | Notes |
|---|---|
| OS | ColorOS 16 (exact build withheld) |
| Device | One realme / OPPO device tested; the method carries over to OnePlus on the same ColorOS generation |
| Prerequisite | Wireless debugging (`adb pair`) plus platform-tools; **no root** |
| Risk | See the notice at the top; the operation is reversible, but you own the consequences |

## Three findings that are easy to miss

1. `pm disable-user` **fails** on packages that have both a factory copy and a store-updated copy — it returns `new state: default`. Use `pm suspend` instead.
2. `suspend` is **per user**: besides `user 0` there are a guest user (user 10) and a multi-app user (user 999), each with its own default-browser role. Check them separately or you will miss one.
3. Whether links still open after disabling depends on where the `BROWSER` role points — **not** on whether the package is disabled.

## License

Documentation is licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/). See [LICENSE](LICENSE).

---

[中文版](README.md)
