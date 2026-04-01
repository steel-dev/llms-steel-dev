---
title: "Mobile Mode for Browser Automation Is Not Just a User-Agent String"
id: "mobile-mode-for-browser-automation"
summary: "Mobile mode in Steel swaps the entire session identity: viewport, touch stack, Chrome build, and fingerprint. It is not just the `User-Agent` header. Use it when the mobile site is cleaner or when traffic must look like a real phone; skip it when the desktop surface is already reliable."
description: "Mobile mode in Steel swaps the entire session identity: viewport, touch stack, Chrome build, and fingerprint. It is not just the `User-Agent` header. Use it when the mobile site is cleaner or when traffic must look like a real phone; skip it when the desktop surface is already reliable."
canonical_questions: ["mobile mode for browser automation is not just a user-agent string"]
intent: "reference"
entity: "browser-automation"
audience: "developer"
schema_type: "Article"
visibility: "public"
ai_visibility: "public"
llms_priority: "core"
token_budget: "medium"
date: "2026-03-31"
updated: "2026-03-31"
related: []
external_refs: []
type: "article"
status: "draft"
canonical_url: "https://steel.dev/blog/mobile-mode-for-browser-automation"
created: "2026-03-31"
modified: "2026-03-31"
tags: [mobile, browser, automation, steel]
immutable: false
---

# Mobile Mode for Browser Automation Is Not Just a User-Agent String

## Short answer
Mobile mode in Steel swaps the entire session identity: viewport, touch stack, Chrome build, and fingerprint. It is not just the `User-Agent` header. Use it when the mobile site is cleaner or when traffic must look like a real phone; skip it when the desktop surface is already reliable.

If you only spoof headers, anti-bot checks still see a desktop browser pretending to be a phone. Steel's `deviceConfig: { device: "mobile" }` creates a mobile-native session that keeps working with [proxies](@/glossary/proxies.md), [CAPTCHA solving](@/glossary/captcha-solving.md), profiles, and embeds, so you get consistent behavior instead of juggling overrides.

### Fit / non-fit snapshot
- Reach for mobile mode when mobile journeys strip complexity: linear checkouts, SMS-only MFA, mobile-first loyalty portals, or responsive flows with different APIs.
- Stay on desktop when you need extensions, large canvas rendering, or hardware features that expect a laptop-class device.

### When to flip the switch
| Pain | What breaks without mobile mode | Steel move |
| --- | --- | --- |
| Checkout hides steps on desktop | Hover menus, multi-column layouts, and hidden controls stall automation | Start sessions with `deviceConfig: { device: "mobile" }` so the UI collapses into one linear form |
| Mobile-only features such as SMS OTP enrollment | Server serves different JS bundles and endpoints by device | Run touching contexts (`hasTouch: true`) and treat the flow as its own script |
| Anti-bot filters catching spoofed UAs | Fingerprint mismatch exposes automation | Use the managed mobile fingerprint that matches viewport, UA, sensor flags, and network timing |
| Token budgets blowing up | Desktop DOM is 2-3x larger than mobile | Send the cheaper mobile DOM to your model or extraction step |

## Instead of spoofing headers, create a mobile session
`deviceConfig: { device: "mobile" }` is the only input you need when creating a Steel session. The session comes back preloaded with a mobile viewport, touch events, and a fingerprint that matches. Keep running Playwright or Puppeteer the same way you do for desktop; you only change the session recipe.

```ts
import Steel from "steel-sdk";
import { chromium } from "playwright";

const steel = new Steel({ steelAPIKey: process.env.STEEL_API_KEY });

const session = await steel.sessions.create({
  deviceConfig: { device: "mobile" },
  useProxy: true,          // optional geo alignment
  solveCaptcha: true       // works the same in mobile mode
});

const browser = await chromium.connectOverCDP(
  `wss://connect.steel.dev?apiKey=${process.env.STEEL_API_KEY}&sessionId=${session.id}`
);
const [page] = browser.contexts()[0].pages();
await page.goto("https://example.com/mobile-checkout");
```

## Implementation path
1. **Choose the flows that gain from linear UI.** Start with journeys that already frustrate headless runs: mobile-first checkout, SMS OTP enablement, banking portals that redirect to `/m/` paths, or property listings that hide filters on desktop.
2. **Create the session in mobile mode.** Set the device config during `sessions.create`. That one flag keeps fingerprint, viewport, and UA aligned, and Steel carries the setting through replays, embeds, and approvals.
3. **Drive it like a phone.** After you connect, either reuse the provided page or spawn a fresh context with `hasTouch: true`, `isMobile: true`, and `viewport` set to a common device profile. Prefer `page.tap()` and `page.swipe()` helpers where they exist. If you stick to `.click`, some elements will ignore you because the JS is watching for touch events.
4. **Treat selectors as distinct contracts.** Mobile templates often collapse ids, rename buttons, or hide nodes until scroll. Store selectors separately and gate them behind a feature flag, not string concatenation. Use Steel profiles if the flow needs authenticated repeats; mobile sessions persist cookies just like desktop ones.
5. **Watch viewport-triggered lazy loads.** Scroll in increments and assert that new cards appear before continuing. Smaller screens defer more content, so combine `page.waitForSelector` with `page.evaluate(() => window.scrollBy(0, window.innerHeight))` loops.
6. **Capture proof from the live viewer or replay.** Use Steel's `debugUrl` to watch the mobile viewport directly or pull an HLS replay when the flow fails. Mobile state is often more fragile, and video saves you from guessing what the phone-sized UI looked like when it broke.

## Trade-offs and guardrails
- Mobile mode is still emulation on a desktop-class machine. Hardware features such as camera uploads, gyroscope gestures, Bluetooth prompts, and push notification enrollment can require a real device for final verification.
- Keep device diversity simple. Start with one tested profile (for example, 390×844 iPhone viewport), then add variants only when a target site behaves differently.
- Real users can still trip server-side rules unique to mobile networks. Combine mobile mode with residential proxies from the same region if you need to mimic subscribers closely.

## When to stay on desktop
| Scenario | Why mobile mode is the wrong tool | Better plan |
| --- | --- | --- |
| Extension-dependent automation | Chrome extensions are desktop only today | Stick with desktop sessions and use the Extensions API |
| High-resolution visual QA | Mobile view hides layout bugs you care about | Automate desktop first, then run mobile as a secondary check |
| CPU-heavy data processing in-page | Mobile bundles throttle timers and degrade performance | Use desktop sessions or run compute on the server after extraction |

## Next steps
1. Light up mobile mode inside your existing Steel quickstart by toggling `deviceConfig` in one workflow.
2. Compare DOM size, token cost, and completion rate versus your desktop baseline.
3. Wire the `debugUrl` or replay into your monitoring channel so you always have evidence of what the "phone" saw.

Humans use Chrome. Agents use Steel.
