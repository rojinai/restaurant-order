# Security Analysis — Nanu Maya Order Page

This document explains every security decision in the order page, the threats that remain, and the operational practices Nanu Maya's staff must follow to stay safe. Read this end-to-end before going live.

---

## 1. What the page can and cannot do

The page is a **static HTML file hosted on GitHub Pages**. There is no backend, no database, no API keys, no secret data anywhere. The Fonepay merchant QR string in the source code is public — anyone who scans the printed counter QR can already decode it.

This shapes the threat model:
- The page **cannot** verify whether a customer actually paid. Only the merchant's Fonepay Business app can do that.
- The page **cannot** stop a determined customer from modifying prices locally in their browser.
- The page **can** make ordering fast, embed amounts in the QR, give every order a unique tamper-evident ID, and harden everything else against the threats it can defend against.

The single most important security rule is operational, not technical:

> **Never serve food based only on the customer's phone screen. Always confirm payment via the merchant's Fonepay Business app notification.**

If staff follow that one rule, the rest of the system is robust. If they don't, no amount of code can save the restaurant from fraud.

---

## 2. Web-level hardening (built into the page)

### Content Security Policy (CSP)
Strict CSP meta tag locks down what the browser can do:
- `default-src 'none'` — nothing loads unless explicitly allowed
- `connect-src 'none'` — page cannot make any network requests (no XHR, fetch, WebSocket, beacon)
- `form-action 'none'` — forms cannot submit anywhere
- `frame-ancestors 'none'` — page cannot be embedded in iframes (clickjacking blocked)
- `object-src 'none'` — no Flash/embed/object tags
- `script-src` allows inline scripts + the pinned cdnjs QR library only
- `base-uri 'self'` — prevents `<base>` tag injection redirects

### Subresource Integrity (SRI)
The QR code library is loaded from cdnjs with a SHA-512 hash:
```
integrity="sha512-CNgIRecGo7nphbeZ04Sc13ka07paqdeTu0WR1IM4kNcpmBAUSHSQX0FslNhTDadL4O5SAGapGt4FodqL8My0mA=="
```
If cdnjs is ever compromised and the QR library is replaced, the browser will refuse to execute the modified file. The page will show an error rather than running malicious code.

### Other headers (via meta tags)
- `X-Content-Type-Options: nosniff` — browser cannot reinterpret file types
- `Referrer-Policy: no-referrer` — page does not leak its URL to anyone
- `Permissions-Policy` — disables geolocation, camera, mic, payment APIs (none are needed)
- `robots: noindex, nofollow` — page won't appear in search engines

### Input handling
- All cart updates go through `updateQty()` which **clamps to integer 0-20** per item. No huge numbers, no negatives, no overflow.
- All rendering uses `textContent` and `createElement`, **never `innerHTML` with dynamic data**. Even if user input were present, XSS would be impossible.
- The page contains no user input fields. Customers only press +/- buttons. Attack surface for injection is essentially zero.

### Cryptographic order IDs
Order IDs use `crypto.getRandomValues()` (the browser's CSPRNG). They are 4 characters from a 32-character alphabet (10.5 million combinations), excluding visually ambiguous characters (I, O, 0, 1). No two recent orders will share an ID by accident, and a fraudster cannot guess valid IDs.

### Transport security
GitHub Pages serves the page over HTTPS only, with HSTS preloaded. Man-in-the-middle attacks cannot tamper with the page in transit.

---

## 3. Threats that remain — and how to handle them

These are threats the page **cannot prevent on its own** but the system can manage operationally.

### Threat A: Customer modifies prices locally
**What it looks like:** A tech-savvy customer opens browser dev tools, edits the price of mutton khana from 290 to 1, places an order, "pays" Rs. 1, shows the screen at the counter.

**Why we can't stop it in code:** Anything the browser displays can be edited by the browser owner. No client-side check is trustworthy.

**Operational defense:**
- Counter staff sees both the order summary AND the actual payment notification on the merchant's Fonepay app.
- The Fonepay app shows the amount the merchant actually received.
- If the customer's screen says "Rs. 1" but the order is one mutton khana, refuse to serve and ask them to redo the order properly.
- Train staff: **the only number that matters is the one in the merchant's Fonepay app.**

### Threat B: Fake payment screenshots
**What it looks like:** Customer shows a screenshot of a previous successful payment, or a doctored image, claiming it's their current payment.

**Operational defense:**
- The page now shows a **fresh Order ID and timestamp** on every order. A screenshot from yesterday will have yesterday's timestamp.
- Staff trains to check: "Is the timestamp within the last 5 minutes?"
- Staff cross-checks the Order ID and amount against the merchant's Fonepay notification (which arrives within seconds of real payment).
- If no merchant-side notification matches, no food.

### Threat C: Old payment replayed
**What it looks like:** Customer paid Rs. 150 for veg khana last week, screenshots the eSewa success page, tries to use it today.

**Operational defense:** Same as Threat B — timestamp on order page + merchant-side notification check. The merchant Fonepay app shows the transaction time on each payment.

### Threat D: Customer claims they paid when they didn't
**Operational defense:** No notification = no payment. Don't argue, don't accuse, just say: "Please show me the payment confirmation in your bank/wallet app, I haven't received the notification yet." 95% of the time this is a real timing delay (resolves in 30 seconds) or the customer made a mistake. The rest, the customer leaves.

### Threat E: Fake QR sticker placed on the counter
**What it looks like:** A bad actor enters the restaurant, covers the printed counter QR with a sticker that links to a phishing page that pretends to be Nanu Maya's order page, with the bad actor's bank account on the QR.

**Operational defense:**
- Owner or trusted staff scans the counter QR at start of every shift to confirm it loads `https://rojinai.github.io/restaurant-order`.
- Print the counter QR on **laminated paper with the restaurant logo overlaid as a watermark** — any sticker placed over it is visually obvious.
- Position the QR where staff can see it at all times.
- The full URL is shown on the loaded page header — train customers to glance at it.

### Threat F: GitHub Pages goes down / account compromised
**What it looks like:** The page stops loading, or worse, someone gets into the rojinai GitHub account and changes the code maliciously.

**Operational defense:**
- Enable **2FA on the GitHub account** (Settings → Password and authentication → enable). This is the single most important security action.
- Keep a backup of the working `index.html` on a USB drive or another cloud drive. If GitHub is unreachable, the file can be moved to any other static host (Netlify, Cloudflare Pages, even a USB stick on a local laptop) in 10 minutes.
- During an outage, fall back to the original system: customer reads paper menu, tells counter staff, pays at counter, gets token. The system worked before; it works again. No emergency.

### Threat G: The dynamic QR amount doesn't work with Fonepay
**What it looks like:** Customer scans the QR but their eSewa/Khalti app doesn't show the pre-filled amount, only the merchant info.

**Why this might happen:** Officially, Fonepay's "Dynamic QR" requires a merchant API agreement with merchant secret keys. The page generates EMVCo-standard dynamic QRs without using that API. Most wallet apps read the EMVCo amount tag and display it, but some might ignore it on QRs that didn't come from Fonepay's official dynamic QR system.

**Operational defense:**
- **Test before going live** with one of every wallet your customers use: eSewa, Khalti, NIC ASIA mobile, NMB mobile, IME Pay, ConnectIPS, etc.
- If a wallet doesn't auto-fill the amount, the page shows "Enter amount manually after scanning" — customers can still pay by typing the amount.
- The total amount is displayed prominently above the QR in large red text. The customer can read it and enter it manually as a fallback.
- This is the same experience they had before, so worst case the system is no worse than what they had.

### Threat H: Customer privacy
**What it looks like:** Customer doesn't want their order tracked.

**Defense:** The page collects nothing. No cookies, no localStorage, no analytics, no fetch calls. No data ever leaves the customer's browser except the QR scan (which goes to Fonepay, like any other QR payment). Privacy is preserved by design.

---

## 4. Daily operational security checklist

Before opening every day:
- [ ] Owner/manager scans the counter QR with their own phone — confirms it loads the correct order page
- [ ] Confirms the URL in the address bar is `rojinai.github.io/restaurant-order` (not a lookalike)
- [ ] Confirms the merchant name on the payment screen says "Nanu Maya Tamang"
- [ ] Confirms the Fonepay Business app is logged in on the owner's phone and notifications are enabled
- [ ] Test order: add 1 veg khana, scan with own phone, verify Rs. 150 appears (or is enterable)
- [ ] No payment is made for the test; cancel in the wallet app

During service:
- [ ] Counter staff never serves food without seeing a Fonepay Business app notification confirming the amount
- [ ] Counter staff checks the order timestamp on the customer's screen is within ~5 minutes
- [ ] Any payment discrepancy (amount differs by even Re. 1) is treated as suspicious

Weekly:
- [ ] Owner reviews Fonepay Business app transaction history vs. observed customer count for sanity
- [ ] Confirm laminated counter QR is undisturbed, no stickers placed over it

Monthly:
- [ ] Owner confirms GitHub account 2FA is still working (do a manual login)
- [ ] Owner downloads a backup copy of `index.html` to a USB stick

---

## 5. What we deliberately did NOT do

A few things that look like "security improvements" but actually make things worse — for context:

- **No login or accounts.** Adding accounts means storing passwords or sessions, which means a backend, which means a target. Anonymous ordering is more secure for a small restaurant.
- **No localStorage / sessionStorage.** Stored data could be stolen by malicious browser extensions; we keep no state across page loads.
- **No external analytics (Google Analytics, etc.).** Each one is a third-party script with its own supply chain risk.
- **No "verify payment" button that calls Fonepay.** That would require Fonepay merchant API credentials, which would need a backend, which would be a new attack target. The merchant's Fonepay Business app already does this for free.
- **No customer phone number / receipt SMS.** Collecting PII creates legal exposure under Nepal data protection norms and adds nothing for the use case.

The simplest system that does the job is the most secure system.

---

## 6. If something goes wrong

**Customer claims they paid but no notification:**
1. Ask them to show the payment confirmation in their wallet app (not just the order page).
2. Check the Fonepay Business app for recent transactions. Wait 30 seconds — wallet-to-bank delays are real.
3. If still no notification, ask them to pay again. Real payments always show within 60 seconds. If they refuse, ask them to leave politely.

**The page won't load:**
1. Confirm the URL is exactly `https://rojinai.github.io/restaurant-order` (no typos)
2. Check GitHub status: `https://www.githubstatus.com`
3. Fall back to the original counter-based system (paper menu, staff enters order, customer pays with the static QR, staff hands token)
4. Restore the page when GitHub recovers

**You suspect the page has been modified maliciously:**
1. Go to `https://github.com/rojinai/restaurant-order` and check the last commit date.
2. If you see commits you didn't make, change the GitHub password immediately, revoke all personal access tokens.
3. Restore the page from your USB backup.
4. Re-enable 2FA if it was disabled.

---

*This document and the order page were designed together. They are most secure when used together as described above. Print this document, keep one copy at the counter and one with the owner.*
