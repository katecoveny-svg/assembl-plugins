# CDR accredited requestor application — Assembl Ltd

> Customer and Product Data Act 2025 · Accreditation class: **Non-intermediary** · MBIE-administered
>
> Application fee (one-off): **$1,500 + GST** · Renewal (annual): **$1,000 + GST**
> Processing time (estimate): **8–12 weeks** · Earliest payments live: **~14 weeks from filing**
>
> Banks at go-live: ANZ, ASB, BNZ, Westpac (live since 1 Dec 2025). Kiwibank payments 1 Jun 2026; Kiwibank customer data 1 Dec 2026.

This is the working checklist for Assembl's CDR application, in the order MBIE will likely ask for it. Tick each item, attach the evidence, file when the bundle is complete.

---

## 1. Legal & corporate

Foundation paperwork. MBIE will check this first.

- [ ] **NZBN** — confirm Assembl Ltd's NZBN registered address matches what we put on the application.
- [ ] **Companies Office records** — pull a current company extract (directors, shareholders, addresses). PDF from companiesoffice.govt.nz.
- [ ] **Registered office address** — must be a physical NZ address. Confirm Assembl's registered address is current.
- [ ] **Directors' identity verification** — government ID (passport or NZ driver licence) for each director. Used for fit-and-proper checks under the AML rules MBIE applies for CDR.
- [ ] **Beneficial ownership disclosure** — anyone with 25%+ of shares or voting rights, disclosed with ID.

## 2. Financial service provider (FSP) registration

Required because Assembl will initiate payments. **One of the two heaviest items.**

- [ ] **FSP registration** — apply via fsp-register.companiesoffice.govt.nz. ~$460 + GST initial, $75/yr ongoing. Allow 4–6 weeks for registration itself.
- [ ] **Dispute resolution scheme membership** — join one of: Banking Ombudsman, IFSO, FDRS, **FSCL** (most common for fintechs). Annual fee ~$1,500.
- [ ] **AML/CFT compliance program** — even as a low-AML-risk reporting entity, the program docs must exist: risk assessment, customer due diligence procedures, suspicious activity reporting policy, training program.

## 3. Insurance & financial coverage

MBIE will not accredit without evidence of adequate coverage against liabilities. Verify exact thresholds against MBIE's published accreditation guidance at filing time.

- [ ] **Cyber liability insurance** — minimum ~$1m, typically more. Quotes from Delta, Quanta, AIG. ~$3k–$8k/yr for Assembl's profile.
- [ ] **Professional indemnity** — ~$1m coverage. Often bundled with cyber.
- [ ] **Public liability** — confirm it covers digital-only activities.
- [ ] **Directors and officers (D&O)** — standard for any FSP-registered entity.

## 4. Security & technical readiness

**The other heaviest item.** MBIE looks at security posture in detail because Assembl will hold consent tokens that can initiate real payments.

- [ ] **Mana Trust Layer must be functioning.** The April 2026 security audit identified **four critical findings** — auth stub, orgId null bypass, absent tier gating, Mana Trust Layer not executing. **All four must be closed before filing.** Bring third-party pen test evidence.
- [ ] **Penetration test report** — independent third-party. NZ providers: ZX Security, Aura Information Security, INSINIA. ~$8k–$15k for fintech scope.
- [ ] **ISO 27001 or SOC 2 evidence** — full cert not required at filing, but a roadmap with target dates is expected. **SOC 2 Type 1** ($15k–$25k) is the more realistic short-term target than ISO 27001.
- [ ] **Data classification & handling policy** — written policy covering how CDR-derived data is classified, accessed, encrypted, transmitted, disposed of.
- [ ] **Encryption** — at rest (Supabase column-level encryption for consent tokens) and in transit (TLS 1.3, certificate pinning for bank API calls). Document current state.
- [ ] **Key management** — where keys live (Vercel env vars at minimum, ideally AWS KMS / GCP KMS via sidecar). Rotation policy.
- [ ] **Access control** — who can access what. Currently Kate is sole; once contractors have DB access, RBAC must be in place.
- [ ] **Audit logging** — every CDR data access and payment initiation goes to an append-only audit log. OpenHands event-stream pattern is the model (typed events, immutable, queryable).
- [ ] **Incident response plan** — written runbook covering Privacy Commissioner breach-notification path for any PII incident.

## 5. Privacy & data handling

- [ ] **Privacy policy** — specific to TŌRO Kid Money. What's collected, from whom, why, how long, who else sees it. Must address kids' data explicitly even though kids never sign up directly.
- [ ] **Privacy Impact Assessment (PIA)** — required for new use of PII at scale. Templates at privacy.org.nz. Review by privacy lawyer (Dentons, Russell McVeagh, Simpson Grierson).
- [ ] **IPP 3A notification flow** — working in-product demo of how parents are notified when child info is collected from third parties (school newsletters, etc.). MBIE may request a demo.
- [ ] **Data Processing Agreements** — signed DPAs from every third party processing PII on Assembl's behalf (Supabase, Vercel, email provider).
- [ ] **Retention and disposal schedule** — already drafted across the three plugin SKILL.md files. Formalise into one policy document.

## 6. Consent flow evidence

- [ ] **Consent UX walkthrough** — screenshots or screen recording of the full parent consent journey: bank selection → redirect to bank → parent authorises → return to TŌRO → kid destination account verification → one-cent test transfer → ongoing payment approval.
- [ ] **Consent text** — exact text shown at each step. Reviewed by lawyer for plain-English compliance.
- [ ] **Consent expiry & refresh flow** — documented and demonstrated. Day 330, day 365, post-expiry behaviour. No payment ever moves on an expired consent.
- [ ] **Withdrawal flow** — parent can revoke consent at any time, immediately, from inside TŌRO. Documented and demonstrated.

## 7. The application itself

- [ ] **Application form** — filed through MBIE's CDR portal once published. Currently in draft on mbie.govt.nz; check at filing time.
- [ ] **Application fee** — $1,500 + GST, paid at submission. Non-refundable.
- [ ] **Cover letter** — two pages max. What Assembl does, what Kid Money does, why we're applying. Include the why-NZ-needs-this framing (first NZ family agent doing direct kid bank payments, koha-built-in, tikanga overlay).
- [ ] **Supporting evidence pack** — all of §§1–6 in a single PDF (or whatever format MBIE specifies). Indexed.

## 8. After filing

- [ ] **Respond to MBIE queries within 5 business days each round.** Slow responses extend the timeline. Expect 1–2 rounds.
- [ ] **Technical onboarding with banks** — each bank requires a 20-business-day technical info exchange + 20-business-day system access window. Plan 8–10 weeks total once accredited.
- [ ] **Sandbox testing at each bank** — run full end-to-end (consent, CoP, payment, audit logging) before going live.
- [ ] **Public registration** — MBIE publicly lists accredited requestors. "Assembl is now an accredited CDR requestor" is a launch-worthy moment.

---

## Indicative timeline

| Week | Activity | Owner |
|---|---|---|
| 1–2 | Close the four Mana Trust Layer security findings | Kate + dev |
| 1–4 | FSP registration + dispute scheme membership | Kate + lawyer |
| 2–5 | Insurance quotes and policies in place | Kate + broker |
| 3–6 | Third-party pen test scoped and run | Pen test provider |
| 4–7 | Privacy policy, PIA, retention schedule drafted and reviewed | Privacy lawyer |
| 5–8 | Consent flow built and documented (Phase 2 of Kid Money) | Kate + dev |
| 8 | All evidence collated into supporting pack | Kate |
| 9 | Application filed with MBIE | Kate |
| 10–18 | Respond to MBIE queries (assume two rounds) | Kate |
| 18–20 | Accreditation granted (best case) | MBIE |
| 20–28 | Bank technical onboarding (parallel across four banks) | Kate + dev + bank teams |
| **28** | **First real payment in production** | Kate |

**28 weeks** from today to first real payment in production. Parallel track: ship Phase 1 of Kid Money (no-CDR, manual-transfer prompt) within 6 weeks — soft-launches the product and validates the chore-and-photo loop while accreditation is progressing. Public Kid Money launch with real CDR payments lines up with week 28, roughly **mid-November 2026**.

## Indicative cost summary

| Item | Estimated cost (NZD, ex GST) |
|---|---|
| MBIE application fee | $1,500 |
| FSP registration | $460 |
| Dispute resolution scheme membership (year 1) | $1,500 |
| Cyber liability + PI insurance (year 1) | $4,000 – $8,000 |
| Penetration test | $8,000 – $15,000 |
| Privacy lawyer (PIA + policy review) | $3,000 – $6,000 |
| SOC 2 Type 1 audit (target within 12 months) | $15,000 – $25,000 |
| **Total to file (excluding SOC 2)** | **$18,460 – $36,460** |
| **Total in year 1 (including SOC 2)** | **$33,460 – $61,460** |

**Funding it.** $33k–$61k year-one outlay is real money. RDTI claim covers a portion of the security and privacy engineering work as eligible R&D — raise with the RDTI adviser at the next quarterly check-in. Callaghan Innovation may also point toward an RDTI credit specific to fintech regulatory compliance. Worth a 30-minute call.

---

**Source:** TŌRO Launch Bundle, 12 May 2026 (files.zip / TORO_launch_bundle.pdf, Part 2).
**The hard parts:** the four security findings (close first), the pen test (book the slot now — they get booked out), the privacy lawyer review (start in parallel with pen test). Everything else is procedural.
