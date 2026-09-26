# Changelog
All notable changes to Splashdown (formerly SettleFlow) are documented here. The engine is
`splashdown/` (MIT library); `saas/` is the thin hosted layer.

## [0.8.0] - 2026-09-26

- **Renamed: SettleFlow is now Splashdown.** Package `splashdown`, CLI
  `python -m splashdown`, repo `AtlasNex/splashdown`, hosted app
  `splashdown.atlasnex.com`. Environment variables are now `SPLASHDOWN_*`; the old
  `SETTLEFLOW_*` names are still read as a fallback, and an existing
  `saas/settleflow.db` is carried over on first start.
- **Fix: bank debits were reported as unexpected credits.** A statement carries money
  out as well as money in. An unmatched debit (rent, salary, a transfer) landed in
  `bank_only`, was exported as `UNEXPECTED_CREDIT` in the Tally CSV, classified
  `DIRECT_TRANSFER`, and netted into the unmatched bank total. Worse, a chargeback
  debit that shared a payout's UTR could be paired with that payout as
  `AMOUNT_MISMATCH` (fake fee drift). Matching is now same-sign only; unmatched debits
  go to a new `ReconResult.bank_debits` list and are never treated as exceptions.
  Regression check added (78 checks).
- **Brand:** the hosted UI moves to the AtlasNex v2 system (Sora / Hanken Grotesk /
  JetBrains Mono, self-hosted under the SIL OFL; Deep Space and Starlight surfaces;
  Splashdown teal). Every colour pair re-computed for WCAG 2.2 AA (`docs/UI-AUDIT.md`).
- Contact addresses are now hello@atlasnex.com and security@atlasnex.com.

## [0.7.6] — 2026-09-13

- **PyPI**: `pip install splashdown` now works — 0.7.5 was uploaded the same day, and 0.7.6
  re-publishes with the corrected README baked in (0.7.5's package metadata still said
  "not on PyPI yet"). The name is reserved by this release.
- **Real email delivery live**: Proton SMTP wired into the hosted service via
  `EnvironmentFile=/opt/splashdown/.env` (mode 600, systemd drop-in on the box). Verified end-to-end: notify → `leads.delivered=1` with the
  message accepted by smtp.protonmail.ch. The pricing page and README no longer carry the
  "install from git / not on PyPI" caveat that is now false.

## [0.7.5] — 2026-09-13

The security-sweep release: every finding from the first real Strix scan
(ATL-242: 1 high, 5 medium, 5 low, 1 info) is fixed, and the hosted UI is
rebuilt to pass WCAG 2.2 AA in both colour schemes.

### Security
- **Rate-limit identity is now deployment-aware** (`pick_client_ip(headers, peer)`):
  `CF-Connecting-IP` is trusted only for an intermediary peer (loopback / private —
  i.e. traffic that came through the Cloudflare tunnel, whose CCI the edge sets and
  a caller cannot forge, D-41); a public peer is keyed on its own socket address, so
  rotating the header no longer mints buckets. IPv6 spellings canonicalize to one
  bucket (vuln-0002).
- **Identity-independent global window** `SPLASHDOWN_GLOBAL_RATE_LIMIT_PER_HOUR`
  (default 600): the total `/reconcile` rate stays bounded even if the identity
  assumption breaks again — the chain-breaking half of vuln-0010's HIGH.
- **`POST /r/{token}/notify` is throttled** (namespaced key) **and its rows are
  pruned** under the advertised 30-day window — it was the only unthrottled write
  (vuln-0001).
- **One-mailbox email validation**: the notify address must round-trip through
  `email.utils.getaddresses` as exactly one unchanged mailbox — a comma no longer
  becomes extra `RCPT TO` recipients (vuln-0011).
- **Derived work is bounded, not just bytes**: `MAX_STATEMENT_ROWS` (200k lines)
  refuses over-wide uploads; `reconcile` moved off the event loop (sync handler +
  FastAPI thread pool); `MAX_CONCURRENT_RUNS` (2) fails fast instead of queuing
  900 MB peaks (vuln-0004). Measured: `/health` worst 594 ms during a 150k-row
  reconcile, previously a 6.3 s service-wide stall.
- **CSV formula injection**: `-` joins the neutralised initiator set; the old test
  asserted a substring and enforced nothing (vuln-0005).
- **SBI netbanking `NameError`** fixed (missing `date` import on the no-year
  fallback branch) (vuln-0007).
- **Money/date/epoch bounds everywhere**: one `_money(token)` guard on every text
  and PDF statement money site; booleans refused by `_paise` like `_rupees` already
  did; `parse_date` rejects non-strings; `_epoch_date` bounded 0..2100; recon
  identity fields must be strings; `csv.Error` arrives as `ValueError`;
  `RecursionError` maps to 400 not 500 (vuln-0003/0008/0009).
- **PDF/OCR work ceilings**: 50 pages / 2 MB decoded content-stream per page /
  2 M chars accumulated, a 40 MP raster guard computed before any allocation,
  per-page temp-dir release, 120 s Tesseract timeout — all surfaced as
  `PdfResourceLimitError(ValueError)` (vuln-0012). The 100-page 39 KB bomb Strix
  measured at 125 s CPU is now refused in <1 ms.

### UX
- Complete CSS token system (`_style.html`): every fg/bg pair passes WCAG 2.2 AA,
  computed; dark theme previously left status colours at 2.5–3.6:1 — all fixed.
- Visible focus rings, styled file-pick buttons, brand mark + favicon +
  OpenGraph share card, `<main>` landmarks. Automated axe-core sweep: 0 violations
  on every page in both schemes.

### Packaging / repo
- pyproject: keywords, full classifiers, project URLs, contact email; `py.typed`.
- README rewritten for launch (screenshots, quickstart, host service reference);
  `CHANGELOG.md`, `docs/UI-AUDIT.md`, `docs/USER-GUIDE.md` added.

### Verification
- Self-check `python tests/test_matching.py`: 74 → 77 checks, all green (CI on
  3.10/3.11/3.12 + zero-dependency + SaaS boot/canary + legal-pages jobs).
- Every fix above has a live gate recorded in `docs/UI-AUDIT.md` and the ATL-242
  board ledger.

## [0.7.4] — 2026-09-11
- Deployed build: whole-request size cap, CCI-only rate-limit identity, run-count
  deploy tripwire, DB-overwrite fix in deploy.sh.

## [0.7.3] and earlier
- Engine, parsers (Razorpay/PayU/Cashfree/PhonePe/Juspay + 7 banks), exports
  (Tally/GST/TDS-1035), CLI, hosted beta.
