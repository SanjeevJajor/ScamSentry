# ScamSentry <img width="128" height="128" alt="icon128" src="https://github.com/user-attachments/assets/68364f02-0fbf-4313-a81a-3b6377a2e802" />
**An explainable scam and malware-redirect shield for Chrome. It warns you before you click, not after you pay.**
Built by team **Hental Mealth** (Computer Applications Hons. Data Science, Chandigarh University) for the **CODE YUDH** hackathon, Open Innovation track.

> **Status:** working prototype, version 0.2.0. It runs fully inside the browser. There is no server of our own and no LLM yet. See <a src="https://github.com/SanjeevJajor/ScamSentry/blob/main/Roadmap"> Roadmap </a> for what comes next.
---
## Why we built this
Online scams in India are growing very fast. Ministry of Home Affairs data, reported by ThePrint, shows about **28.15 lakh cybercrime complaints in 2025** (24% more than 2024) and about **₹22,495 crore lost**, with investment scams causing roughly three-quarters of it.
Existing defences have gaps:

| Existing defence | Gap |
|---|---|
| Safe Browsing / VirusTotal style blocklists | A brand new scam domain is not listed until someone reports it |
| Antivirus | Many scams are only text ("pay ₹10 to verify your KYC"), so there is nothing to scan |
| 1930 helpline and cybercrime.gov.in | Help starts only after the money is gone |

ScamSentry warns **at the moment of reading**, and always says **why**.

---

## What it does

- **Scores every page from 0 to 100**, shown on the extension badge (✓ for trusted sites).
- **Explains the score.** Every point has a plain-English reason, so users also learn what a scam looks like.
- **Shows a warning banner** on risky pages (high only, caution or high, or never).
- **Checks the address:** raw IPs, look-alike (punycode) characters, `@` tricks, brand names on the wrong domain (a "SBI" page that is not sbi.co.in), fake `gov` sites, risky endings like `.xyz`, hyphen-stuffed names and free website builders.
- **Reads page text on your device** with rules for Indian scams: registration fees, "earn ₹X daily" tasks, Telegram/WhatsApp job chats, OTP/UPI PIN requests, KYC and digital arrest threats, fake prizes, unrealistic returns, fake virus warnings, fake browser updates and APK lures.
- **Checks domain age** through RDAP. A domain registered 2 days ago is a strong warning sign.
- **Checks a public phishing feed** and, optionally, the URLhaus malware list, and can **block known malware sites** before they load.
- **Tracks the redirect chain** ("how did I get here?") and flags automatic redirects, many hops, shorteners and known-bad hops.
- **Trusted sites list** to silence warnings for sites you know are fine.

---

## How it works

```
 Page loads
     │
     ▼
 content script ── reads page text on your device ──► rule counts only (no text leaves the tab)
     │
     ▼
 service worker (background.js)
     ├─ url-analyzer   → address red flags
     ├─ text rules     → scam wording (counts sent from the tab)
     ├─ rdap           → domain age (only the domain name is sent)
     ├─ feed / malware → known-bad lists (checked locally)
     └─ redirects      → how you reached the page
     │
     ▼
 scoring.js → score + list of reasons → badge, popup, warning banner
```

Domain age is looked up in a second step. A first verdict appears quickly from the cache, then is refined and pushed to the page when the lookup finishes.

### How the score is built

Every point is a visible reason. Examples from `lib/scoring.js`:

| Signal | Points |
|---|---|
| Listed in a public phishing feed or malware list | 100 |
| Domain registered under 30 days ago | 35 |
| Brand name used on an unofficial domain | 35 |
| Link to a known malware site | 40 |
| Sent here by a known bad site | 40 |
| Domain under 6 months old | 15 |
| Domain older than 2 years | −10 |
| Scam wording (fee demand, KYC threat, etc.) | by rule weight, text total capped at 70 |

Levels: **0–24 Looks safe**, **25–54 Be careful**, **55–100 High scam risk**.

To avoid crying wolf:
- Awareness articles that talk about scams count much less.
- Big well-known sites are scored more gently.
- Text alone is capped, so a pile of weak words cannot push a page to the top.
- A site you mark as trusted is always 0.

---

## Install (developer mode)

1. Download or clone this repository.
2. Open `chrome://extensions` in Chrome 116 or newer.
3. Turn on **Developer mode** (top right).
4. Click **Load unpacked** and select the folder that contains `manifest.json`.
5. A welcome settings page opens. Visit any website and check the badge on the ScamSentry icon.

**Optional, for malware blocking:** make a free abuse.ch account, copy your URLhaus Auth-Key and paste it in the extension settings. The key stays in your browser and is only sent to abuse.ch.

---

## Privacy

- Page text is analysed **inside the tab**. Only rule IDs and counts go to the background worker. Nothing goes to any server of ours (there isn't one).
- The only network calls are RDAP (the registrable domain name only), the public phishing feed download, and the URLhaus download if you add a key.
- Redirect tracking is observe-only. It keeps host names per tab in session storage, cleared when the tab closes.
- Page-derived text is always shown with `textContent`, never as HTML.

---

## Permissions and why we need them

| Permission | Why |
|---|---|
| `storage` | Settings, trusted sites, cached domain ages and downloaded lists |
| `alarms` | Refresh the feeds every 6 hours |
| `webRequest` | Watch page navigations to build the redirect chain (never blocks or changes requests) |
| `declarativeNetRequest` | Block known malware sites with a warning page |
| `http(s)://*/*` host access | Scan pages you visit |

---

## Project structure

```
manifest.json          Manifest V3 config
background.js          Service worker: scan pipeline, message router, feeds, redirect tracking
content/
  text-rules.js        Scam wording rules (regex) and weights
  content.js           Reads the page, collects signals, handles single-page apps
  banner.js            Warning banner in a closed Shadow DOM
lib/
  url-analyzer.js      Address heuristics (pure functions)
  scoring.js           Turns all signals into score + reasons (pure)
  rdap.js              Domain age lookup with cache and rate limiting
  feed.js              Public phishing feed parsing and lookup
  malware.js           URLhaus list + blocking rules
  redirects.js         Redirect chain tracking and scoring
  lists.js             Bundled lists (brands, free hosts, risky endings, shorteners)
popup.html, popup/     Popup: current verdict and reasons
options.html, options/ Settings page
blocked.html, blocked/ Page shown when a malware site is blocked
shared/tokens.css      Shared design tokens
icons/                 Extension icons
```

---

## What we learned

### Chrome extensions (Manifest V3)
- **The service worker can stop at any time**, so nothing lives only in memory. State goes in `chrome.storage` and listeners are registered at the top level.
- **Content scripts and the service worker are separate worlds.** They talk only by messages, so each message has a version and operation name, and the worker checks who sent it.
- **Never trust a content script blindly.** The worker validates and clamps everything, and accepts a page's own URL only if its origin matches what Chrome reports.
- **Single-page apps do not reload.** For sites like LinkedIn we watch URL changes and use a MutationObserver (with a rescan limit) to catch late-loading content.
- **`declarativeNetRequest`** blocks known-bad sites without reading every request, but rules have limits, so the list is chunked and capped.
- **A closed Shadow DOM** stops page CSS and scripts from restyling or reading our banner.

### Detection and scoring
- **Explainable beats mysterious.** "35 points because the domain is 2 days old" is more useful than a bare percentage, and it teaches the user.
- **Combine, cap and scale weak signals.** No single regex decides the verdict, and awareness articles must not look like scams.
- **False positives are the real enemy.** A tool people stop trusting is worse than none, so we added allowlists, trusted sites, gentler scoring for big sites and a CAUTION level before HIGH.
- **Never flag a whole platform.** For a phishing page on a shared site like a form builder, only the exact URL is listed.
- **Domain age is a strong, cheap signal.** Scam sites are usually brand new, so it catches threats blocklists have not seen.
- **Brand look-alikes need care.** We compare against the real registrable domain, so `jobs.flipkartcareers.com` is fine but `flipkart-careers.xyz` is not.
- **Redirect chains tell a story.** We tell a click from an automatic redirect by checking whether the user interacted with the previous page.

### Privacy and security
- **Do sensitive work on the device.** Reading text locally and sending only counts made privacy a design feature.
- **Treat page text as hostile.** It is inserted as plain text, and "Continue anyway" appears only in a real top-level tab so a website cannot trick users through a frame.
- **Keep secrets out of logs and UI.** The URLhaus key is stored locally and never appears in pages or error messages.

### Engineering and teamwork
- **Be polite to free APIs.** RDAP is rate limited, so we added caching, a persisted rate limiter, a timeout and backoff on 429 errors. If a lookup fails, the tool still gives a verdict from what it has.
- **Keep the logic pure.** Address analysis, scoring and redirects have no `chrome.*` calls, so they are easy to test.
- **Agree on the interface first.** Fixing the data format between parts let everyone build in parallel with mock data.

---

## Known limitations

- Rules are regex and heuristics, so they can miss new wording and misjudge unusual pages. It is a warning aid, not a guarantee. Stay careful with links and payments.
- It scans web pages you visit, not messages inside separate apps (WhatsApp mobile, SMS).
- Domain age needs the registry to answer, so some endings return no data.
- The feed and malware list are only as fresh as the last refresh.
- Chrome only for now.

---

## Roadmap

**Next**
- AI risk engine: an LLM that names the scam type and adds a plain-English reason to the same explainable score.
- A scan API and community blocklist, so a confirmed report protects the next person.
- Report portal with a threat dossier page and a paste-anything scanner.

**Later**
- Hindi and Hinglish scam detection
- QR code and screenshot scanning (OCR)
- Firefox and Edge versions
- Android app for SMS and notifications
- Pre-filled complaints for cybercrime.gov.in and 1930

---

## Team Hental Mealth

Sagar · Rachit · Sanjeev · Sunayana

## Data sources

- Phishing feed: [phishunt.io](https://phishunt.io)
- Malware list: [URLhaus by abuse.ch](https://urlhaus.abuse.ch)
- Domain registration data: [RDAP via rdap.org](https://rdap.org)
- Statistics: Ministry of Home Affairs data, as reported by [ThePrint](https://theprint.in/india/cybercrime-saw-24-spike-in-2025-indians-lost-rs-22495-crore-mainly-in-investment-scams/2859930/)

## License

ScamSentry is free software, released under the [GNU General Public License v3.0](LICENSE). You can use, study, change and share it, but any version you distribute must also be released under GPL-3.0 with its source code.

---

*A hackathon prototype, not affiliated with any bank, government body or the services listed above.*
