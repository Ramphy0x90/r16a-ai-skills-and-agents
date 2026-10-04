---
name: privacy-legal-compliance
description: Use before launching a web app, mobile app or self-hosted service that handles personal data, to check required privacy disclosures and draft a reviewable policy. Covers GDPR, Swiss FADP, app-store labels, E2EE messaging.
---

# Privacy & legal compliance for shipping an app

**This skill is not legal advice and does not substitute for a lawyer.** It exists to catch the common failure mode — publishing with *no* privacy policy or disclosure at all because "it's just a personal project" — and to produce a reviewable draft. Any real launch with real third-party users (even unpaid, even friends-and-family scale) should have an actual lawyer review the final text before it goes live, especially for anything end-to-end encrypted or EU-facing. Say this plainly when using this skill; don't present a drafted policy as final or legally sufficient.

## The core misconception to correct first

**"It's a personal/hobby project, not a business" does not exempt it.** GDPR and the Swiss FADP both apply based on *what data is processed and whose*, not based on revenue, company status, or project size. The moment a service handling personal data (even just an email + password, even just an IP address in logs) has a user who isn't the developer themselves, disclosure obligations can apply. A single self-hosted instance with five friends using it is still "processing personal data" under both frameworks.

## When a privacy policy / disclosure is required

- Any account system (email, username, password, profile data).
- Any message/file content stored on a server you control, even if encrypted.
- Any logging that captures IP addresses, device identifiers, or user agents (most web servers do this by default).
- Any analytics, crash reporting, or third-party SDK (even a "free" one — it's still a data processor).
- Any cookie or local-storage use beyond strictly necessary session state.
- Any app store listing (Apple App Store and Google Play both *require* a privacy policy URL to even submit an app, regardless of what the app does).

If none of the above apply (a truly offline, no-account, no-telemetry tool), a privacy policy may genuinely not be required — don't force one onto a project that collects nothing.

## GDPR-aligned disclosures (applies if any user is in the EU/EEA, regardless of where you or your server are)

A privacy policy should state, in plain language:
- **Who is the data controller** — your name/entity and a contact method (an email is sufficient for a personal project).
- **What data is collected** — be specific and exhaustive: account fields, message/file metadata (even if content is E2EE and unreadable to you, metadata like timestamps, sender/recipient, file size often isn't), IP/log data, device info.
- **Why it's collected** (purpose) and **the legal basis** — for a self-hosted service a user opts into, this is usually "performance of a contract" (providing the service they signed up for) or "legitimate interest" (basic security logging); avoid claiming "consent" as the basis for things that aren't actually optional.
- **How long data is retained** — be concrete (e.g., "message content is retained until the user deletes it; server logs are retained for 30 days").
- **Who else sees it** — any third party or sub-processor (a cloud provider, an email-sending service, a crash-reporting SDK) must be named or at least categorized.
- **Where it's processed** — if your server is in Switzerland/EU, say so; this matters for the next point.
- **International transfers** — if any processor (e.g., a US-based email API) receives EU user data, note the transfer mechanism (Standard Contractual Clauses, adequacy decision, etc.) — don't skip this if you're using any US-based third-party service.
- **User rights** — access, rectification, erasure ("right to be forgotten"), data portability, and the right to lodge a complaint with a supervisory authority. For a self-hosted single-user-controlled service, "how to exercise this" can be as simple as "email me" or "use the in-app delete-account function."
- **Breach notification posture** — you don't need a polished incident-response plan for a hobby project, but the policy shouldn't claim guarantees you can't back up (avoid "we guarantee your data is 100% secure").

## Swiss FADP (revised Federal Act on Data Protection, in force since Sept 2023) — if the controller or any users are in Switzerland

Similar transparency obligations to GDPR but with real differences — don't just copy a GDPR policy and call it FADP-compliant:
- No GDPR-style enumerated "legal basis" requirement in the same structured way, but purpose limitation, transparency, and data minimization are still required principles.
- A Swiss-based controller processing data of Swiss residents is squarely in scope regardless of EU exposure.
- Cross-border transfer rules exist but are structured differently from GDPR's adequacy/SCC mechanism — check the current Federal Data Protection and Information Commissioner (FDPIC) guidance rather than assuming GDPR's mechanism applies verbatim.
- If the service has both Swiss and EU users, the policy generally needs to satisfy both frameworks — in practice this usually means writing to the stricter of the two requirements on each point rather than maintaining two separate policies, unless there's a reason to split them.

## App store requirements (for mobile apps)

- **Apple App Store**: requires a working privacy policy URL before submission, and requires filling out the "App Privacy" (nutrition label) questionnaire in App Store Connect — declaring every data type collected (even if not sold/shared) and whether it's linked to identity, used for tracking, etc. Mismatches between the declared label and the app's actual behavior are a common rejection/removal reason — the questionnaire answers must match the privacy policy must match what the code actually does.
- **Google Play**: requires a privacy policy URL and a completed "Data safety" section in Play Console with equivalent disclosures (data types collected/shared, encryption in transit, whether deletion is supported).
- Both stores additionally require a justification string for sensitive runtime permissions (camera, contacts, location, etc.) — the justification shown to the reviewer/user should match what the feature actually needs, not be broader "just in case."
- For an E2EE messaging app specifically: both stores' export-compliance questionnaires ask about encryption — answer accurately; standard end-to-end encryption of user content typically falls under either an exemption or a straightforward self-classification, but the exact classification can change, so check current App Store Connect / Play Console guidance at submission time rather than relying on this skill's text indefinitely.

## E2EE / messaging-specific regulatory notes (for E2EE/federated messaging systems)

- The core legal/technical story for an E2EE service is usually: "we cannot read message content; we only hold what's structurally necessary to route it (and even that may be minimized)." State this precisely and accurately in the policy — don't overclaim "we can't see anything" if any metadata is in fact visible server-side.
- The EU has ongoing regulatory attention on E2EE messaging and content-scanning obligations (commonly referred to as "Chat Control" proposals) — this is an actively evolving area, not settled law, and the regulatory outcome affects what an E2EE provider can truthfully claim and may affect what's legally required of them in the EU market. Don't draft permanent policy language that assumes the current landscape is final; flag this specific area as one to re-check close to any EU launch rather than treating it as a one-time compliance check.
- This skill does not provide, and should not be asked to provide, guidance on circumventing lawful-intercept or content-moderation obligations — only on accurately disclosing the service's actual architecture and data handling.

## How to use this skill

1. Identify what the project actually collects/processes — read the data model (DB schema, request/response shapes), check for third-party SDKs (analytics, crash reporting, push notification services), and check logging configuration for what's captured.
2. Walk the relevant sections above (web-only vs. mobile-and-store vs. E2EE-messaging-specific) against that actual inventory — don't draft generic boilerplate without first knowing what the app does.
3. Draft a plain-language policy covering every applicable disclosure above, using concrete specifics from step 1 rather than vague placeholders where the actual answer is known.
4. Clearly mark the output as a **draft for legal review**, not a final published policy — in the document itself, not just in your own output to the user.
5. Flag anything genuinely uncertain (cross-border transfer mechanism, current app-store classification requirements, the Chat Control regulatory status) as "verify before publishing" rather than guessing confidently.
