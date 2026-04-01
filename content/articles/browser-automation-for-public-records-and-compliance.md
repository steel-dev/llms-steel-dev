---
title: "Browser Automation for Public Records and Compliance Workflows"
id: "browser-automation-for-public-records-and-compliance"
summary: "Show legal and compliance teams how Steel sessions keep court dockets, FOIA pulls, and filings automated with stateful profiles and audit-ready evidence."
canonical_questions: ["browser automation for public records and compliance workflows"]
intent: "reference"
entity: "browser-automation"
audience: "developer"
schema_type: "Article"
visibility: "public"
ai_visibility: "public"
llms_priority: "core"
token_budget: "medium"
date: "2026-04-01"
updated: "2026-04-01"
related: []
external_refs: []
type: "article"
status: "draft"
canonical_url: "https://steel.dev/blog/browser-automation-for-public-records-and-compliance"
description: "Show legal and compliance teams how Steel sessions keep court dockets, FOIA pulls, and filings automated with stateful profiles and audit-ready evidence."
created: "2026-04-01"
modified: "2026-04-01"
tags: [public-records, compliance, vertical-workflow]
immutable: false
---
Public records and compliance teams only trust automation when every run looks like a deputized clerk: authenticated entry, predictable state, and an evidence packet on release. Chronicle Legal already runs more than 100,000 government-portal sessions a month on managed browsers to keep disability cases fresh, and Benny saw a 40 percent lift in successful government benefit lookups once [CAPTCHA solving](@/glossary/captcha-solving.md) and [proxy](@/glossary/proxies.md) control moved out of their scripts. Those volumes highlight why the runtime, not just the agent plan, decides whether these workflows scale.

Instead of wiring Playwright or Selenium to a rotating cast of desktop VMs, treat Steel as the managed browser tier: sessions spin up in under a second, stay live for up to 24 hours, keep portal trust inside persistent profiles, and release with replays and agent logs teams can file with counsel. You still own the case logic, but Steel supplies concurrency ([Steel Local](@/glossary/steel-local.md) handles roughly one session, [Steel Cloud](@/glossary/steel-cloud.md) starts in the hundreds), stealth, CAPTCHA tooling, and the audit-friendly plumbing that general-purpose headless browsers skip.

## Where compliance teams lose hours
- Dockets, licensure boards, and FOIA portals bounce between SSO, SMS codes, and idle timers. Without persisted profiles every retry becomes a password reset and a potential security finding.
- Portals dump downloads into opaque temp folders or drive-by popups. Evidence disappears unless workflows mount a predictable `/files` path and archive outputs before anyone closes the browser.
- Manual review blocks everything. Investigators want a live view before approving a filing, then expect immutable replay plus agent logs when the case hits discovery.
- Queue spikes melt flaky browser farms. Local fleets collapse after a dozen concurrent logins or whenever Windows Update reboots the host. Steel Cloud lets you request hundreds of sessions at once while managed proxies and CAPTCHAs keep logins steady.

## Workflow map
| Workflow | What usually breaks | Steel pattern |
| --- | --- | --- |
| Court docket sweeps and legal notice watchlists | Multi-tenant logins get rate limited, per-county portals expire in minutes, and operators cannot show what the bot actually saw | Create tagged sessions per jurisdiction, reuse `profileId` so trust cookies survive, schedule `sessions.release()` plus HLS replay export at the end of each sweep |
| Public records pulls and FOIA exports | Downloads land on random disks, evidence trails get lost, and FOIA queues time out while you stitch data together | Upload request payloads via Files, mount `/files` for download targets, and ship the archive plus agent logs to case storage as soon as the run completes |
| License renewals and compliance filings | OTP delivery, notarized attachments, and per-agency credentials force humans back into the loop | Store credentials with namespaces, include `totpSecret` when available, keep attachments in Files so agents can feed webforms, and gate `debugUrl` access for approvals |
| Investigations and evidence packages | Opposing counsel asks how the data was collected and who touched it, long after plan retention windows expire | Capture metadata on session creation, mirror replays and logs into your storage before plan deadlines, and attach viewer activity records to case notes |

## Recommended browser pattern
1. **Tag every run on creation.** Pass case identifiers, jurisdiction, and workflow type into `metadata` so every replay, log, and file is traceable when subpoenas land months later.
2. **Seed profiles per portal.** Manually log in once with `persistProfile: true`, finish MFA, then reuse the `profileId` until the portal forces a reset or the 30 day inactivity timer hits. Keep each profile under the 300 MB cap by clearing downloads after every batch.
3. **Namespace credentials.** Upload per-agency secrets into the Credentials API, require namespace matches in session creation, and delete secrets automatically when an operator leaves the program.
4. **Bundle attachments through Files.** Intake scans, affidavits, and receipts belong in Files so agents can upload via deterministic paths and your system can download one archive when the session ends.
5. **Wrap the live viewer.** Proxy `debugUrl` behind SSO, default `interactive=false`, and log every escalation to ensure approvals have a human signature.
6. **Export evidence on release.** Chain HLS replay downloads, agent log export, and Files archive mirroring immediately after `sessions.release` so nothing ages out of plan retention.
7. **Budget concurrency.** Steel Local is perfect for development and on-prem enclaves but caps concurrency near one session. Steel Cloud starter tiers begin in the tens of concurrent sessions and Pro pushes into the hundreds with managed proxies and CAPTCHA coverage, so pick the plan that matches your docket velocity.

## Trust and audit controls
| Requirement | Steel surface | How to wire it |
| --- | --- | --- |
| Portal identity | Profiles + Sessions metadata | Store a profile per portal and inject `metadata` so every browser action maps to a case and user persona |
| Credential custody | Credentials API | Keep logins vaulted, enforce namespaces, inject TOTPs when available, and audit access per secret |
| Attachment hygiene | Files API | Move uploads and downloads through Files so nothing sits on developer laptops and every document inherits case IDs |
| Live oversight | `debugUrl`, embeds, agent logs | Serve the viewer through your ACL, log interactive handoffs, and pair each replay with the agent log export for counsel |
| Evidence retention | HLS + Files exports | Export replays, logs, and archives before the plan retention window, then store them in your compliance bucket |
| Data residency | Steel Cloud vs Steel Browser | Use Steel Browser in your own VPC when statutes require in-boundary processing, otherwise lean on Steel Cloud for managed proxies and CAPTCHA solving |

## Works for / Not yet
**Works for**
- Monitoring court dockets, board agendas, and enforcement bulletins across dozens of portals that share a Chromium front end
- Running FOIA queues, licensure renewals, and compliance filings that need attachments plus auditable evidence packages
- Investigations where counsel demands video proof of what the automation accessed and when approvals were granted

**Not yet**
- Desktop-only court software or terminals that never expose a browser surface Steel can attach to
- Hardware token flows without SMS or TOTP fallbacks
- Scenarios where you cannot store replays, logs, or files outside the portal, because auditors still expect exported evidence

## Next step
Pick one docket sweep, seed a profile, upload the agency credentials under a namespace, and run it through Steel Cloud so you can capture replays, agent logs, and a Files archive in a single queue item. The docs that matter are [docs.steel.dev/overview/sessions-api/overview](https://docs.steel.dev/overview/sessions-api/overview), [docs.steel.dev/overview/profiles-api/overview](https://docs.steel.dev/overview/profiles-api/overview), [docs.steel.dev/overview/credentials-api/overview](https://docs.steel.dev/overview/credentials-api/overview), and [docs.steel.dev/overview/files-api/overview](https://docs.steel.dev/overview/files-api/overview).

Humans use Chrome. Agents use Steel.
