# Privacy Policy v3 — Boilerplate Cross-Check Findings

**Date:** 10 May 2026 (overnight pass, session 1)
**Purpose:** Compare v3 against the Australian Privacy Principles (APPs), Apple App Store requirements, Google Play requirements, and conventions from published privacy policies of indie subscription apps. Identify gaps, recommend additions, preserve voice.

---

## TL;DR

**v3 is solid in voice, structure, and content.** The plain-English approach is genuinely good and aligns with the OAIC's own guidance ("clearly expressed... easy to understand, avoiding jargon, legalistic and in-house terms"). Most published indie privacy policies are far worse than this draft.

**Five things are genuinely missing** that need to be added before going live:

1. Data security / how the data is protected
2. Data breach notification (Notifiable Data Breaches scheme — required if you ever cross AU$3M turnover, but worth including from day one as a trust signal)
3. International data transfers (Cloudflare's global infrastructure means user data crosses borders — APP 8 requires disclosure)
4. International users (GDPR for EU, CCPA for California — you're shipping globally)
5. Account deletion process clarity (Apple has a hard requirement here that needs explicit acknowledgment)

**One thing should be re-checked when the time comes:**
- The "what we collect" list will need a final pass against the Apple Privacy Nutrition Label categories when submitting to App Store Connect (they have specific category language). Not a v4 issue, a Phase 5 issue.

**Three smaller polish items** worth incorporating to v4:
- Be explicit that you're an APP entity (small business, technically below the $3M threshold, but voluntarily applying the Privacy Act to PICKLE MOON)
- Add a "Last reviewed" date alongside "Last updated" — different concepts that matter for compliance
- Clean up the "globally distributed" line — APP 8 requires more specific disclosure about overseas data transfer

**Nothing legally-required is wrong in v3.** The gaps are additions, not corrections.

---

## What v3 has right

These match published indie privacy policies that successfully pass App Store review:

- **APP 1 (transparent management):** v3 clearly explains who you are, what you collect, why, and how to contact you. ✓
- **APP 5 (notification of collection):** v3 lists every field collected, in plain language. Stronger than 90% of policies in this respect. ✓
- **APP 6 (use and disclosure):** v3 names every party who can see the data (you, the partner, Cloudflare, Apple/Google). ✓
- **APP 11 (security obligations):** v3 *references* security via "we don't share/sell" but doesn't describe the actual security posture. Gap — see Finding 1.
- **APP 12 (access to personal information):** v3 covers this in "Your rights." ✓
- **APP 13 (correction of personal information):** v3 covers this in "Your rights." ✓
- **OAIC complaints pointer:** v3 includes the correct OAIC contact. ✓
- **Children's policy:** v3 has the COPPA-style 13+ section. ✓ (Apple specifically reads this section.)
- **Plain English voice:** OAIC's own guidance literally asks for this. v3 nails it.

The voice and structure are not just "fine" — they're *better than the published policies of most indie apps*. Don't let any of the additions below dilute that.

---

## What needs to be added to v4

### Finding 1 — Data security section

**What's missing:** v3 doesn't describe how user data is protected. APP 11 requires reasonable steps to protect personal information from misuse, interference, loss, unauthorised access, modification, or disclosure. The privacy policy should describe these steps at a high level — not technically, but enough to demonstrate a security posture exists.

**Why it matters:** Apple reviewers look for this. App stores in 2025-2026 are scrutinising security claims more aggressively. Saying "we secure your data" without saying *how* reads as hand-wave.

**What to write (high level):**

> ## How we protect your data
>
> All data sent between Four Tasks and our servers travels over encrypted connections (HTTPS/TLS). Your data is stored on Cloudflare's infrastructure, which uses industry-standard encryption at rest. Access to the database is restricted to The Pickle Moon — no one else has a key.
>
> We follow the principle of least data: we don't ask for what we don't need, and we don't store what we don't need.
>
> No system is perfectly secure. If we discover a data breach that's likely to harm you, we'll let you know directly and notify the Office of the Australian Information Commissioner (OAIC), as required by Australian law.

This is short, honest, and gives reviewers something concrete. The "principle of least data" line reinforces v3's "what we don't collect" section, which is a brand-aligned signal.

---

### Finding 2 — Data breach notification (NDB scheme)

**What's missing:** v3 doesn't acknowledge the Notifiable Data Breaches scheme.

**The legal nuance:** Strictly, the NDB scheme only applies to organisations with annual turnover above AU$3 million, or in specific industries (health, financial, etc.). PICKLE MOON is well below that threshold for years. So *legally*, you're not obligated to comply.

**However:** Even at sub-$3M turnover, the OAIC strongly encourages voluntary compliance, and most well-written small-business privacy policies include it. The reasoning: it's a trust signal, it costs nothing to commit to, and it future-proofs you. If you cross $3M turnover (5000 paying users at $4/month is ~$240k/year — still well below, but the path is up), you're already compliant.

**What to write** (folded into Finding 1's section above):

> If we discover a data breach that's likely to harm you, we'll let you know directly and notify the Office of the Australian Information Commissioner (OAIC), as required by Australian law.

This single sentence covers it. Don't write a longer treatment — it would be disproportionate.

---

### Finding 3 — International data transfers (APP 8)

**What's missing:** v3 says "Cloudflare's infrastructure is globally distributed." That's accurate but not specific enough for APP 8, which requires disclosure of overseas data transfer.

**The legal nuance:** APP 8 requires that if personal information is disclosed to an overseas recipient, the privacy policy must (a) name the countries where the recipient is likely to be located, or (b) at least state that data may be transferred overseas and describe the safeguards.

For Cloudflare: data centres are in 200+ cities worldwide. Listing every country isn't realistic. The convention is to say "globally" but be honest that this includes outside Australia.

**What to write** (replacement for v3's "Where your data is stored" section):

> ## Where your data is stored
>
> Four Tasks stores data on infrastructure provided by **Cloudflare**, which operates a globally distributed network. Your data may be stored at any Cloudflare data centre worldwide, including locations outside Australia.
>
> We don't sell, lease, or share your data with any third party. Cloudflare hosts the data on our behalf — they're a hosting provider, not a partner. Cloudflare's own data handling is governed by their privacy policy, available at cloudflare.com/privacypolicy/.
>
> By using Four Tasks, you acknowledge that your data may be stored and processed outside Australia. The Pickle Moon takes reasonable steps to ensure that any overseas storage handles your data consistently with Australian Privacy Principles.

The "By using Four Tasks, you acknowledge..." line is a small piece of legal armour required by APP 8. It's a single sentence and stays in voice.

---

### Finding 4 — International users (GDPR / CCPA)

**What's missing:** v3 doesn't address users outside Australia. Four Tasks will be on the Apple App Store and Google Play, both global. Some users will be in the EU (GDPR), some in California (CCPA), some elsewhere.

**The legal reality:** GDPR applies to any business processing personal data of EU residents, regardless of where the business is located. CCPA applies to California residents under similar logic. Both are real exposures.

**The good news for Four Tasks:** Because you collect so little, your GDPR/CCPA obligations are mostly already covered by the v3 content. You just need to *acknowledge* these laws and confirm you respect them.

**What to write** (new section near the end):

> ## If you're outside Australia
>
> Four Tasks is available worldwide. Wherever you live, the same principles apply: we collect what's listed above, store it on Cloudflare, share it with no one, and delete it when you ask.
>
> If you're in the **European Union or United Kingdom**, you have additional rights under the General Data Protection Regulation (GDPR), including the right to data portability and the right to lodge a complaint with your local data protection authority. To exercise any of these rights, email **thepicklemoon@gmail.com**.
>
> If you're in **California**, you have rights under the California Consumer Privacy Act (CCPA), including the right to know what we hold about you, request deletion, and opt out of the sale of your data. We don't sell data, so the third applies by default. Email **thepicklemoon@gmail.com** to exercise the others.
>
> The legal basis we rely on for processing your data is the performance of our contract with you (you've installed the app, we run the app, that requires storing your data) and your consent (you typed it in).

Three short paragraphs. Covers the major international regimes without making the policy feel legalistic. The last paragraph is GDPR-required (Article 13) and is actually quick and clean.

---

### Finding 5 — Account deletion process clarity (Apple hard requirement)

**What's missing:** v3 mentions the reboot button as the deletion mechanism, which is good. But Apple's specific requirement (since June 2022) is more detailed than this. Apple wants:

- **In-app account deletion** must be a clear, findable feature — not buried in settings or requiring an email to support.
- **All personal data associated with the account** must be deleted, not just the account itself.
- **The deletion timeline** should be specified (e.g. "immediately" or "within 30 days").
- The process must be documented in the privacy policy.

v3 covers most of this implicitly but not explicitly. Apple reviewers verify this section against actual app behaviour during review.

**What to write** (replacement for v3's "How long we keep your data" section):

> ## How long we keep your data, and how to delete it
>
> Your data stays on our servers for as long as you use the app. There are three ways it can be deleted:
>
> 1. **You delete it from inside the app.** Tap the **reboot button**, confirm, and your record is permanently removed from our servers immediately. There's no soft delete and no archived backup — once you reboot, the data is irrecoverable, by design.
> 2. **You ask us to delete it.** Email **thepicklemoon@gmail.com** with the username you used. We'll delete your record within 7 days and email you to confirm.
> 3. **You cancel your subscription and stop using the app.** Your record stays where it is unless you also use one of the methods above. We don't proactively delete inactive users at this stage.
>
> When we delete your record, we delete everything associated with it — task labels, completion data, status messages, coin counts, settings, bug reports, and subscription state. We don't keep "soft-deleted" copies for any reason except where the law specifically requires us to (which is currently nowhere — but if that ever changes, this section will be updated).

This is more verbose than v3's original treatment but it's the correct trade-off for a hard Apple requirement. The numbered list keeps it scannable.

---

## Smaller polish items

### Polish 1 — Be explicit about APP entity status

Add a single line near the top:

> The Pickle Moon is a small Australian business. Although we're below the AU$3 million annual turnover threshold that automatically makes a business an "APP entity" under the Privacy Act 1988, we voluntarily apply the Australian Privacy Principles to how we handle your data. Everything below describes what that means in practice.

This is a strong move. It signals you've actually thought about the legal framework and chosen to comply rather than dodging the threshold. App store reviewers and privacy-aware users both notice.

### Polish 2 — Add a "Last reviewed" date

Replace the existing date line with:

> **Last updated:** 10 May 2026
> **Last reviewed:** 10 May 2026

These are different concepts. "Last updated" = when content changed. "Last reviewed" = when it was last checked for accuracy without a content change. Apple reviewers and EU regulators expect both. You'd update "Last reviewed" annually even if nothing changes.

### Polish 3 — Subscription section: a small fix

In the v3 "Subscriptions and payment" section, the line:

> "Apple and Google handle that part directly"

is fine but could be sharper. Suggest:

> "Apple and Google handle the entire transaction. Your payment details go to them, never to us. We receive only a notification telling us whether your subscription is active."

Slightly more precise about the data flow. Same voice.

### Polish 4 — Acknowledgment of subscription metadata

Subtle — but the subscription system doesn't *just* tell us active/not-active. It also tells us:

- Renewal date
- Subscription tier (in case future tiers exist)
- Originating store (Apple vs Google)
- Whether the subscription is in a free trial period
- Whether the subscription is in grace period (lapsed payment)

These need to be in the "what we collect" list because we genuinely will store them. v3 mentions some but not all. Suggested updated bullet for "what we collect":

> - **Subscription state** — whether your account currently has an active subscription, the renewal date, which app store the subscription came from (Apple or Google), and whether you're currently in a free trial or grace period. We do not see card details, billing addresses, or any other payment information — Apple and Google handle that part directly.

---

## What does NOT need to be added

These come up in some boilerplate templates but **don't apply** to Four Tasks:

- **Cookie banner / cookie policy** — only relevant for web. Native apps don't use cookies.
- **Third-party SDKs disclosure** — you have none.
- **Advertising / ad networks** — explicitly chosen against in monetisation position. Stay silent on this in the policy.
- **Analytics disclosures** — none used.
- **Push notifications** — deferred to v1.x in the roadmap. Add to the policy when implemented.
- **Microphone / camera / location permissions** — none requested.
- **Sale of data** — explicitly not happening; v3 already states this.
- **Children's online services certification** — only relevant if app is targeted at children, which yours isn't.
- **Affiliate or marketing relationships** — none.

The absence of these is a *feature* of Four Tasks's privacy policy, not a gap. Don't pad the policy with sections explaining what you don't do (beyond the existing "What we don't collect" section, which is justified).

---

## Voice notes for v4

Going from v3 to v4 will add roughly 400-500 words. To keep the voice, the writing approach should be:

- **Don't add headers without content.** Every new section earns its space.
- **Keep sentences short.** v3 averages ~15 words per sentence. v4 should match.
- **Numbers, not letters, for ordered lists** (matches existing style).
- **Bold the keywords** the user is scanning for ("reboot button", "thepicklemoon@gmail.com", "subscription"). v3 already does this well.
- **Don't use "we may" weasel language.** v3 uses concrete present tense ("we store", "we don't share"). Keep that.
- **End every section with something the user can actually do.** v3 does this — the "Your rights" section ends with the OAIC link, the contact section ends with the email. Continue this.
- **End the doc with the closing line** that's already in v3:

> That's it. If anything in here is unclear, email us and we'll explain it (and probably rewrite the unclear bit).

Don't lose this line — it's voice signature.

---

## What to do next

I'm writing v4 now, immediately following this findings doc. v4 will incorporate every recommended addition above, in voice, in the right structure. You'll have one document to read tomorrow rather than two.

If you want to push back on any of the findings (e.g. you don't want the international users section because you'd rather geo-restrict the app to Australia), tell me and we'll adjust v4 accordingly.

Otherwise — read v4 fresh, tell me what doesn't sound like you, we land v4.1 if needed, then this tile is done.
