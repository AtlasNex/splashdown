# Decisions

The why behind the design, one entry per decision. Numbers are stable and are
referenced from code comments and other docs; gaps in the sequence are internal
process decisions (naming, hosting, tooling) that do not affect how the library
behaves.

### D-2: Open-core + thin SaaS, not a platform
- **Why:** the enterprise recon *platform* layer is crowded (Gini, UnPay, ReconPe,
  Cointab, Paxcom). The un-owned thing is the *component layer* (parsers + matching).
  OSS core = trust + distribution; the paid layer is a thin SaaS + consulting.

### D-3: `Decimal` for money, never `float`
- **Why:** reconciliation math must be exact; `float` produces rounding drift
  (₹0.1 + ₹0.2 != ₹0.3). Non-negotiable.

### D-4: Two-pass matching (exact UTR -> amount+date fallback)
- **Why:** the real Razorpay flow uses `settlement_id` as the batch key, and the bank
  statement's UTR is the *correspondent bank's* UTR, not the gateway's `settlement_utr`.
  So UTR match alone under-matches; the (amount, date) fallback is what actually pairs
  most batches.
- **Source:** Razorpay `Fetch All Settlements` docs + Terra Insight reconciliation guide.

### D-5: `settlement_id` rides through as `Txn.ref`
- **Why:** `match_settlements` converts `Settlement` -> `Txn` with `ref=settlement_id`,
  so a matched/unmatched result still tells you *which batch* without a separate mapping.

### D-6: paise -> rupee conversion lives only in the parser
- **Why:** Razorpay returns money in the smallest currency unit (paise). Keeping the
  model in rupees and converting at the parser boundary means the matcher never has to
  know about paise.

### D-7: Generic CSV loader with explicit column mapping (no vendor-specific classes)
- **Why:** the Razorpay CSV column names were not verifiable in raw docs (JS-rendered),
  and fabricating a schema is exactly the failure mode that broke Nova's zoning layer
  (20 fabricated `(GUESS)` legend rows). So `load_csv` takes column names as explicit
  arguments; the vendor mapping is data, filled from a real sample, never guessed.

### D-8: Assert-based self-check, no pytest/tox
- **Why:** ponytail rule: non-trivial logic leaves one runnable check, no frameworks.
  `python tests/test_matching.py` is the whole suite. Add pytest only with a recorded decision.

### D-10: MIT license
- **Why:** maximally permissive for an open-core strategy (OSS core MIT, paid SaaS on
  top). Copyright holder = AtlasNex.

### D-14: Phase 2 schema verified via Wayback snapshot of the Razorpay recon docs
- **Why:** Razorpay's live docs are JS-rendered (no schema in raw HTML) and Firecrawl
  was billing-blocked. The June-2026 Wayback snapshot of
  `razorpay.com/docs/api/settlements/fetch-recon/` served the full page, yielding the
  **24 documented response parameters** of `GET /v1/settlements/recon/combined` and the
  verbatim sample response. This is the verified schema for `parse_razorpay_recon`, not
  a guess (D-7 satisfied).

### D-15: Recon parser is strict — unknown fields raise, never silently ignored
- **Why:** the parser validates against the known key set and raises on unknown or
  missing-required fields. If Razorpay adds a field, the parser fails loudly instead of
  dropping data — the failure mode that corrupted Nova's zoning layer was silent
  fabrication, so the opposite default (fail closed) is deliberate.

### D-16: Vendor/bank formats are a registry of column maps, data not code
- **Why:** Phase 3 (Cashfree/PayU/PhonePe/Juspay + bank statements) cannot be built
  without real sample files; their CSV headers are not verifiable from JS-rendered docs.
  The registry (`splashdown/schemas.py`) ships the loader + empty slots that raise a
  clear "needs a real sample" error instead of shipping a fabricated schema. This is the
  correct D-7 posture: the *mechanism* is done, the *data* is DEFERRED pending real
  samples. Never marked as done.

### D-17: The SaaS layer (saas/) may use FastAPI/uvicorn/Jinja2; the library stays stdlib-only
- **Why:** CONSTRAINTS.md #6 (zero runtime deps) protects the library's installability
  and the "MIT, zero-dependency" positioning. The thin SaaS is a separate optional
  layer (`pyproject` `[project.optional-dependencies].saas`), so the core's zero-dep
  invariant is preserved while the paid layer can use FastAPI + sqlite3 (stdlib).

### D-19: Phase 3 column maps sourced from public evidence, not private accounts
- **Why:** the library is for other merchants' files, so no single private account can
  define the formats. They are
  verifiable from public sources: official sample files (Razorpay's xlsx samples still
  hosted on razorpay.com/docs), official docs (Cashfree/PhonePe/Juspay field tables),
  and open-source parsers that read the exact real exports (bank fixtures). Two research
  subagents cross-corroborated every header from ≥2 independent sources. Wired: Razorpay
  settlement CSV (7 cols) + recon CSV (27 cols), HDFC/SBI/ICICI/Axis/Kotak statements.
  NOT wired (honest): Cashfree recon (two-section file), PhonePe (undocumented type/date
  values), Juspay (unstated money unit), PayU (user-configurable columns). Full headers
  + sources in `docs/SCHEMAS.md`.
- **Structural change:** bank statements use TWO columns (debit/credit), so a
  `BankColumnMap` and `load_bank_statement_csv` were added (credits positive, debits
  negative, preamble rows auto-detected). `parse_date` extended for `dd/mm/yy`,
  `dd MMM yyyy`, `dd-MMM-yyyy`, and ISO/datetime-with-time.
  **Date:** 2026-08-16.

### D-20: Bank statements arrive as PDF too — PDF parser is a separate layer
- **Why:** SBI statements usually arrive as PDF, not CSV. The library currently has no PDF
  path. PDF extraction is a different trust boundary (layout parsing, password, possible
  OCR) and must be built against a REAL file (D-7), never a guessed layout. `pypdf` and
  `pymupdf` are already installed, so no new dependency. It waited on a real
  statement layout.

### D-21: SBI PDF parser v1 = YONO text layer only; netbanking + OCR deferred
- **Why:** the parser needed a real SBI layout. Following D-19 (public evidence, not
  private accounts), the real SBI statement text layouts were sourced from the
  Apache-2.0 `raptar231/indian-bank-statement-parser` fixtures (already
  anonymised), so the parser is built against real layouts, never a guess (D-7).
  SBI has two statement layouts: the modern YONO/e-statement table
  (`Date | Transaction Reference | Ref.No./Chq.No. | Credit | Debit | Balance`)
  and the legacy netbanking table (`Txn Date | Value Date | Description |
  Ref No./Cheque No. | Debit | Credit | Balance`). v1 parses YONO only; the
  netbanking PDF's day/month/year values split across wrapped lines in the text
  layer (and its CSV export is already wired via the `sbi` bank map), so the
  parser raises `PdfLayoutError` rather than half-parsing it. Scanned PDFs (no
  text layer) raise `PdfScannedError` — OCR is a separate, unbuilt layer.
- **Structural change:** new `splashdown/pdf.py` (D-20's separate layer) with
  `extract_pdf_text` (lazy pymupdf import; password + scanned detection),
  `parse_sbi_statement`, `parse_sbi_pdf`. pymupdf is an OPTIONAL dependency
  (`[project.optional-dependencies].pdf`), so CONSTRAINTS #6 (stdlib-only core)
  still holds. Fixtures vendored under `tests/fixtures/sbi/` with a NOTICE.md
  (Apache-2.0 attribution). Version bumped 0.4.0 -> 0.5.0.

### D-22: Kotak Dr/Cr combined-amount statement — auto-detected
- **Why:** Kotak's netbanking statement uses a SINGLE combined amount column
  with an explicit Dr/Cr marker ("347.00 Dr" = debit, "35,000.00 Cr" = credit)
  — a different layout from the two-column variant A already wired. The marker
  makes the sign unambiguous in flattened text, so it is parseable without
  coordinates (unlike PNB's separate Withdrawal/Deposit columns, whose empty
  cells collapse ambiguously — deferred). Real anonymised fixture sourced from
  `raptar231/indian-bank-statement-parser` (the D-19/D-21 precedent).
- **Structural change:** `splashdown/parsers.py` gained `parse_drcr_statement`
  (a trailing "amount Dr|Cr balance" trio, narration accumulated across wrapped
  lines) + `load_bank_statement_drcr`. `load_bank_statement` now auto-detects
  the Dr/Cr format (header carries "(Dr)" and "(Cr)") and routes to it, so
  `bank="kotak"` works for either variant. PNB is deferred (ambiguous flattened
  text — needs coordinate-aware extraction or the raw PDF).

### D-23: Complete bank coverage + LLM triage + CLI (end-to-end)
- **Why:** "complete all the things end to end." Applied the D-19 "get the files
  yourself" precedent to source real anonymised fixtures for PNB, DBS, SBI
  netbanking and SBI credit card from `raptar231/indian-bank-statement-parser`
  (Apache-2.0), and adopted its proven technique: recover the debit/credit sign
  from running-balance arithmetic when separate Debit/Credit columns collapse in
  the extracted text (this reverses D-22's earlier "PNB deferred" call).
- **Added:** `parse_bank_text` (generic collapsed-column parser with
  narration-before vs narration-after) -> PNB + DBS wired via TEXT_BANK_PARSERS;
  `parse_sbi_credit_card` (Date|Description|Amount + trailing Cr marker);
  `parse_sbi_netbanking` (legacy "Txn Date|Value Date|..." with the year split
  onto a "yyyy yyyy" line); `parse_sbi_statement` now auto-dispatches
  YONO/netbanking/credit-card; `triage_exceptions` (provider-agnostic LLM hook,
  `call_llm(prompt) -> str`, core still makes no network call); a CLI
  (`python -m splashdown reconcile -> tally.csv + exceptions.csv`).
  `load_bank_statement` now content-routes Dr/Cr, PNB/DBS and SBI text.
  Reference numbers (UPI/UTR/12+ digit) are extracted into Txn.utr.
- **Deferred (honest):** Cashfree/PhonePe/Juspay settlement files (no public
  sample exists; their schemas are already captured in SCHEMAS.md from official
  docs); scanned/image-only PDFs (OCR is a separate unbuilt layer); the hosted
  SaaS (later stage). Kotak "bankii" variant B still needs a real sample.

### D-24: Gateway settlement files are merchant-private; verified schemas + Kotak bankii-B wired
- **Why:** the gateway formats needed real samples. Exhaustive public search (vendor docs, GitHub repo+code search, the
  vetted bank-fixture repos raptar231 and jasimmk/bankii, and real production
  reconcilers aravindsiva13 + SaiYadav1818/Karatly) CONCLUSIVELY found NO public
  sample FILE for the Cashfree/PhonePe/Juspay/PayU settlement or recon CSVs, nor
  IDFC. Every vendor report is generated from an authenticated Merchant Dashboard
  (and contains real transaction/PII data), so these files are never published
  the way anonymised bank-statement text is. D-19's "get the files yourself"
  precedent therefore does NOT extend to them; the gate is real, not a search gap.
- **Verified from real sources (not guessed, D-7):** Kotak "bankii" variant B
  schema transcribed from the production parser `jasimmk/bankii/in_kotak.py`
  (`Serial | Transaction date | Value date | Description | Chq / Ref No. |
  Debit amount | Credit amount | Balance | Dr/Cr`, `%d-%m-%Y`); Cashfree orders
  report (13 cols, from a production `CashfreeSettlementCsvParser`); PhonePe 14
  fields + Juspay 25 cols (already captured in SCHEMAS.md from official docs).
- **Wired:** Kotak bankii-B as a `BankColumnMap` + content auto-detect in
  `load_bank_statement` (header tokens "Debit amount"/"Credit amount"), so the
  existing `kotak` key auto-handles a bankii export. The Dr/Cr flag is redundant
  (only one amount column populated per row). Self-check 38 -> 39.
- **Deferred (honest):** parse-from-real-file verification for bankii-B (needs a
  real Kotak bankii CSV export; D-7 no synthetic fixtures); Cashfree/PhonePe/
  Juspay/PayU settlement+recon parsers (need a real dashboard export from any
  merchant account); IDFC; scanned/OCR PDFs; hosted SaaS. UNBLOCK = one real
  dashboard export per gateway (the exact list is in `docs/SCHEMAS.md`).

### D-25: Rigorous stress test found + fixed 2 real bugs; documented ceilings
- **Why:** a full stress test before release.
  Ran two adversarial harnesses (34 + 16 checks) against the invariants in CONSTRAINTS:
  Decimal money, deterministic matching (incl. 500-line fuzz), trust boundaries, expose
  /classify/parse edge cases, Level-2 netting + order matching, hostile-input fuzz, and a
  real generated PDF.
- **Bug 1 (fixed):** exports emitted money inconsistently — a whole-number amount printed
  as `1000` while a fractional one printed `1000.00`, and an exempt TDS printed `0` vs
  `0.00`. For accountant-facing CSVs (Tally/GST/TDS) that's a real defect. Fixed with a
  single `_money()` helper (quantize to `0.01`) applied to every money cell; all existing
  '100.00'/'-2446.84'/'1.00' asserts stay green.
- **Bug 2 (fixed):** `extract_pdf_text` silently accepted a NON-PDF file. pymupdf
  leniently opens plain text as a 1-page doc, so the library returned the raw text instead
  of failing — the "silently coerce" anti-pattern CONSTRAINTS #2 forbids. Fixed by guarding
  on `doc.is_pdf` after open and raising a clear `ValueError` (D-7 trust boundary).
- **Verified ceilings (kept, not fixed — documented):** (a) `match()` (amount,date)
  fallback can mis-pair when two bank lines share (amount,date) — already in CONSTRAINTS;
  (b) `classify()` DUPLICATE_SUSPECT does not decrement the bank-side count, so it can
  over-flag (conservative; flagged for human review, so safe); (c) `_epoch_date` converts
  Razorpay's UTC epoch to a UTC date, which can skew the (amount,date) fallback by one day
  for settlements near midnight UTC (IST is +5:30; primary UTR match unaffected). Fixing
  (c) switches `created_at` to IST and would ripple through tests/docs — deferred for a
  deliberate call, not a silent change.
- **Self-check:** 39 -> still 39 (no regression); both stress harnesses green.

### D-26: Fix the remaining found gaps — IST epoch, date format, CLI PDF gating
- **Why:** close every gap the D-25 stress test reported.
- **(1) `_epoch_date` now returns the IST (+05:30) date.** Razorpay's epoch is UTC;
  Indian bank statements are IST. The UTC date was one calendar day behind IST for any
  transaction within ~05:30 of midnight UTC, which skewed the (amount, date) fallback
  (the primary UTR match was unaffected). Now aligns with the statement's IST date.
- **(2) Date parser accepts `%d-%b-%y`** (e.g. "01-Jul-25") — added after `%d-%b-%Y` so
  a 4-digit year still matches first; no ambiguity.
- **(3) CLI `.pdf` gating:** `--bank` != sbi with a .pdf statement now raises a clear
  error ("PDF parser only handles SBI") instead of running the SBI parser and throwing a
  confusing SBI-layout error.
- **Deliberately KEPT (not bugs):** `_is_money`'s exactly-2-decimals requirement — widening
  it to accept integers would let a numeric reference (e.g. YONO "500") be misread as a
  money amount and break real statements; it's a disambiguator, not a bug. `classify()`
  DUPLICATE_SUSPECT over-flagging stays conservative (better to over-flag for human review
  than under-report genuine duplicates).
- **Self-check:** 39 -> 41 (2 new: IST epoch + two-digit month name; CLI PDF gating).
  Version 0.7.1 -> 0.7.2.

### D-27: OCR for scanned SBI PDFs (Tesseract) + SaaS deploy-ready
- **Why:** scanned (image-only) statements are common, and the hosted service needed a
  repeatable deploy path. Tesseract is used as a system binary rather than a new OCR stack.
- **OCR (splashdown/ocr.py, optional [ocr] extra):** rasterises each page with pymupdf and
  runs the Tesseract CLI via subprocess (--psm 6). The recovered text feeds the existing
  `parse_sbi_statement` dispatcher. On a clean scan Tesseract preserves the statement's
  columns/header, so the native parser works (verified: 2 rows recovered). Added
  `ocr_pdf_text`, `parse_sbi_scanned_pdf`, `OcrUnavailableError`, `OcrError`;
  `parse_sbi_pdf(path, ocr=True)` auto-falls back on `PdfScannedError`; CLI `--ocr` flag.
  Honest ceiling: OCR recovers a text layer, not perfect columns — a noisy scan can merge
  columns and the parser raises `PdfLayoutError` rather than guessing. The `[ocr]` extra is
  just pymupdf (Tesseract is a system binary, invoked via CLI, no pip OCR package).
- **SaaS deploy-ready:** added `Dockerfile` (python:3.11-slim, install `.[saas]`, run
  uvicorn) + `docker-compose.yml` (binds 127.0.0.1:8091:8000, container_name splashdown) +
  `.dockerignore`. Verified the app boots locally and `/health` returns
  {"status":"ok","version":<library version>} (fixed a stale hardcoded "0.3.0" to use
  `splashdown.__version__`). Actual live host deploy is a separate infra step (Cloudflare
  Tunnel/port), handled outside this repo.
- **Self-check:** 41 -> 42 (OCR test, guarded to skip if tesseract/pymupdf absent).
  Version 0.7.2 -> 0.7.3.

### D-29: A run is reached by an unguessable token, not an account and not a listing
- **Why:** `/runs` listed every uploaded statement to anyone, and `/runs/<id>/export/tally.csv`
  downloaded the workpaper. Integer ids made it enumerable, so the exposure was one `for` loop
  wide. Any real merchant's first upload would have been world-readable.
- **What:** `secrets.token_urlsafe(24)` (~144 bits) per run; results at `/r/<token>`, exports at
  `/r/<token>/export/<name>.csv`. No accounts, no login — that would be a different product and
  the thinnest thing that closes the hole is a capability URL.
- **Consequence accepted:** no account means no recovery. Lose the URL and the run is gone. The
  results page says so in as many words ("Bookmark this page") rather than pretending otherwise.
- **Legacy rows:** predate the column, keep a NULL token, and are unreachable. That is the intended
  outcome, not a migration gap.

### D-30: Run responses are `no-store`, and the canary asserts it
- **Why (evidence, not theory):** after the route was removed, the live site still served the leaked
  CSV from Cloudflare's edge — `cf-cache-status: HIT`, `Age: 2079`, `max-age=14400`. Reminiscent of
  the real lesson: **removing a route does not un-publish what a CDN already holds**. `.csv` is in
  Cloudflare's default cacheable-extension list.
- **What:** middleware sets `Cache-Control: no-store, private` and `X-Robots-Tag: noindex` on
  `/r/*`, `/reconcile`, `/health` and `*/notify`. `/health` is included because a cached health
  check is the "process is up" illusion. The canary's `cache_headers` check fails if this regresses.
- **Verification:** through the public URL — `/r/<token>` → `cf-cache-status: DYNAMIC`,
  export → `BYPASS`; the previously cached URL now 404s. Purge was by API `prefixes`
  (`<host>/runs/`).

### D-31: The origin host is never committed
- **Why:** `scripts/deploy.sh` had `root@<origin-ip>` as a default. The repo is now public and that
  address is deliberately absent from public DNS (Cloudflare proxies the hostname), so publishing it
  hands out a route that bypasses the Cloudflare rules in front of the origin. Two docs named it too;
  both were redacted.
- **What:** the host comes from `SPLASHDOWN_HOST` or a gitignored `scripts/.deploy.env`. `--help`
  works with no target set; the scripts refuse to run without one and say how to set it.
- **Side benefit:** the deploy scripts are now genuinely usable by anyone self-hosting, which the
  hardcoded address made impossible.

### D-32: A canary does a real reconcile, because a health check cannot see a broken product
- **Why:** the failure mode this project keeps hitting is "process up, product broken" — a container
  heartbeating for weeks, a dashboard rendering blank, an app answering 200 on `/health` while the
  results page is wrong. A liveness probe is blind to all of it.
- **What:** `scripts/canary.py` (stdlib only) uploads a real settlement file and bank statement,
  follows the private run URL, reads the tally export, asserts every path from the 11 Sep incident is
  still 404, and asserts run responses are still `no-store`. It runs in CI (proving the app boots on
  a clean runner), on the server after every deploy, and every 15 minutes from an external watchdog
  against the public URL.
- **Design choice:** the checks live in the repo, not the watchdog, so they move with the product.
  The watchdog only does alerting (first failure, every 6th, recovery).

### D-33: D-24's "merchant-private" conclusion was wrong for PhonePe, and PayU was the wrong shape
- **What D-24 said (2026-08-24):** Cashfree/PhonePe/Juspay/PayU settlement and recon files are
  merchant-private dashboard exports; no public sample is obtainable; D-7's gate therefore blocks
  all four.
- **What the 2026-09-11 search found (docs/RESEARCH-gateway-samples.md):** two real committed
  PhonePe merchant exports with 1,038 data rows (which also resolve the undocumented `PaymentType`
  values and date formats D-19 was blocked on); a real Cashfree recon export proving 63 columns,
  not 48; two official PayU settlement **APIs** with complete documented sample JSON; and a
  readily-fetchable Juspay docs page with the full 25-column schema.
- **Why D-24 was wrong:** it searched for a *CSV* for PayU, and no CSV can exist — PayU's export
  columns are chosen per merchant in a dashboard dialog, so the correct target was always the API.
  For PhonePe it appears no one had looked at committed merchant exports in unrelated repositories.
- **Consequence:** PhonePe is now wired (with the per-settlement netting the file shape requires);
  PayU's fix is API parsers; Cashfree recon needs a two-section parser; IDFC remains genuinely
  deferred — for IDFC the gate held up.
- **Lesson for this repo:** "not publicly available" is a conclusion that expires. Re-test a
  blocking claim before treating it as a constraint, and record *what was searched*, not just the
  verdict — the negative results in the research doc are as useful as the positives.
- **Also decided:** the real files must not be vendored (they carry a named merchant's real UTRs);
  fixtures are regenerated from the real header with invented values, as D-19 did for banks.

### D-34: The Cashfree recon file is read whole — two sections, two maps, and the flag applies to the SETTLEMENT amount
- **Why:** the export is literally two reports concatenated: a 14-column batch section, the marker
  line `** Settlement Reconciliation Details **`, then a 63-column event section. Reading either
  through the other's map states wrong money confidently, so `load_cashfree_recon_report()` splits
  on the marker first and raises when it is absent.
- **The non-obvious part, and the bug it produced:** the event section has no debit/credit pair.
  Direction comes from `Sale Type ∈ {CREDIT, DEBIT}` and the amount it applies to must be
  `Event Settlement Amount` — the money movement, net of the fee/tax columns beside it and printed
  NEGATIVE on a refund — while `Event Amount` is the gross. The first implementation applied the
  flag to the gross, so netting the event section did not equal the batch total: it was out by
  exactly the fees (5900.00 vs 5881.12 on the fixture). The fix is a separate
  `direction_amount_col`; the check that catches a regression is the fixture assertion that the
  netted event total equals the batch total, which is verified to fail when the column is wrong.
- **Also decided:** the movement is taken as a magnitude and the flag sets the side, so a vendor
  printing a refund as `-100.00` and one printing it `100.00` land identically; and the `UTR` is
  both the line's settlement_utr and its batch key, because the event section carries no batch id.
- **Consequence:** `cashfree_recon_csv` (63 cols) and `cashfree_settlement_csv` (14 cols) are both
  filled, replacing two `None` placeholders.

### D-35: PayU's settlement CSV is closed as un-wireable BY DESIGN; the two APIs are the surface
- **Why:** PayU's own docs say the export's columns are chosen per report in a dashboard dialog, so
  no fixed public header can exist. Every earlier "no sample file found" conclusion about PayU was
  right in outcome and wrong in reasoning — the search could not have succeeded, and recording it
  as a search gap is what made the next session search again.
- **Instead:** `parse_payu_settlement_range` (UTR-level → `Settlement`: `settlementAmount` is the
  net credited to the merchant's bank account and `utrNumber` is the bank reference, so these rows
  reach `match()` at level 1 rather than through the amount+date fallback) and
  `parse_payu_transaction_details` (per transaction → `ReconLine`; `settlementAmount` arrives as a
  JSON number and SIGNED, so the magnitude goes to `amount` and the sign to debit/credit).
- **Fail-closed, deliberately:** PayU's `status: 1` envelope raises with PayU's own message, and an
  unknown field raises. A vendor adding a field is a schema change, and ignoring it silently is how
  a reconciler ships wrong numbers after a release nobody connected to the vendor.
- **Rejected:** leaving `payu_settlement_csv` open "awaiting a sample". It can never be filled.

### D-36: Juspay is wired with its limits written into the loader, not left awaiting a perfect file
- **Why:** the 25-column schema is officially documented and Juspay's own production parser names
  the columns it reads, so refusing to build would preserve the blocker, not the correctness.
  Nothing here is guessed: every column name is transcribed from one of those two sources.
- **What is still NOT verified, stated in the loader docstring and in SCHEMAS.md:** the money unit
  (the docs say only "Integer" while Juspay's own parser reads plain rupee decimals → rupees, a
  corroboration rather than a documented fact); one `Settlement Date` being one bank credit for the
  documented variant, which has neither a batch id nor a UTR column to key on; and the status
  filter (only `Settled` means funds reached the bank).
- **Consequence:** a Juspay reconciliation is a run that needs review until a real file and its
  matching bank credit agree. An unrecognised `Settlement Status` raises instead of being silently
  included or dropped, because a row filtered out by a guess is an error nobody ever sees.
- **Rejected:** a `juspay_recon_csv` map from the HyperPG column set. It has no real file behind it
  either, and two speculative maps is twice the unverified surface with none of the evidence.

### D-38: Strix is the independent security gate, and a gate that finds issues must FAIL

- **Decision.** The Strix OSS CLI runs in `.github/workflows/security.yml` as this project's
  independent security gate, on `pull_request`, on `push` to `master` (the branch we actually deploy
  from — `pull_request`-only was watching a door nobody uses), and on `workflow_dispatch`. Exit
  code **2 (vulnerabilities found) fails the build**; artifacts are uploaded on both paths so a
  failing gate is always actionable.
- **Why the exit-2 branch is load-bearing.** The first version only echoed a `::warning` and fell
  through. The first real scan then concluded **success** while its artifact held 12 findings
  (1 high, 5 medium). A gate that finds issues and reports success is worse than no gate: it
  manufactures false assurance, and it teaches everyone to ignore CI. The rule is general — a
  security gate's pass must mean *no findings*, never *the job didn't error*.
- **Why an independent scanner at all.** It found a latent `NameError` in a shipped SBI parser, a
  `Date`-boolean-to-money path, and CSV formula injection surviving a leading `-` — none of which
  our own 74-check self-check or the in-session manual review caught. The self-check asserts what we
  thought to assert; an adversarial scanner tests what we didn't.
- **Consequence.** A green security run is a claim about *what was analysed*, not a statement that
  the project is secure — and it must be read via the artifacts, not the conclusion. The artifact
  expires after 30 days, so findings get a durable copy outside the repo.

### D-41: The CF-Connecting-IP premise is settled — CCI is unforgeable through the tunnel; the invariant to keep is tunnel-only ingress

- **Decision.** Recorded from a live probe: the Cloudflare edge rejects (403, error code
  1000) any request carrying a client-supplied `CF-Connecting-IP` before it reaches cloudflared, and
  sets the header itself. `pick_client_ip` (CCI-only) is therefore sound **on this topology**, and
  vuln-0010's HIGH chain does not close from outside. The binding invariant: the origin keeps its
  127.0.0.1 bind and tunnel-only ingress — a future direct exposure re-arms the finding.
- **Why it still matters.** XFF passes through with the caller's value as the FIRST hop (origin
  echo-proved), so live v0.7.3's XFF-keyed limit is bypassable today; this decision raises deploy
  urgency with evidence. It also re-ranks the security findings: vuln-0002's work-bound fix is the top item, not
  the identity fix.
- **Evidence:** a probe record kept outside the public repo (it describes the ingress, not
  the origin address).

### D-42: Rate-limit identity is deployment-aware (peer evidence + global window); the PDF/OCR layer gets explicit work ceilings

- **Decision (identity).** Strix's suggested fix for vuln-0002 gated the trusted header on a
  LOOPBACK peer — correct for the live topology (verified on the box: splashdown.service runs
  plain uvicorn on 127.0.0.1:8093, cloudflared connects from loopback), but it breaks the repo's
  own documented Docker path: with userland port-mapping the container-side peer of EVERY
  connection is the bridge gateway (172.17.0.1) — private, constant, never loopback — and a
  loopback-only gate would collapse all clients into one shared bucket there. `pick_client_ip
  (headers, peer)` therefore trusts `CF-Connecting-IP` only when the peer is an INTERMEDIARY
  (loopback or RFC1918/link-local) — the address D-41 proved the edge sets — and keys a PUBLIC
  peer on its own socket address, which its owner cannot mint per request. IPv6 spellings
  canonicalize to one bucket. `SPLASHDOWN_GLOBAL_RATE_LIMIT_PER_HOUR` (600) adds the
  identity-independent window (vuln-0010's chain-breaking half): the per-identity key can be
  minted away, this cannot.
- **Decision (work bounds).** The upload caps bound bytes, not work. `MAX_STATEMENT_ROWS` (200k
  lines — ~30x the largest real CA statement; a compact-JSON payload cannot exceed ~190k items
  inside the same 10MB cap, so one check bounds both sides) refuses over-wide uploads with a clear
  400; `reconcile` became a synchronous handler (FastAPI thread pool — the event loop stays
  responsive: health worst 594ms during a 150k-row run, previously a 6.3s full stall) under a
  2-slot semaphore (3 concurrent heavies -> 303/303/429). The PDF/OCR layer (library-only, not
  reachable from the hosted route) gets: 50 pages, 2MB decoded content-stream per page (stored
  compressed length never trusted), 2M accumulated text chars, 40MP raster ceiling computed BEFORE
  get_pixmap (MediaBox is author-chosen), per-page temp-dir release, and a 120s tesseract timeout
  surfaced as `PdfResourceLimitError(ValueError)`. Residual, stated: one page's parse cost cannot
  be bounded inside the process — untrusted-file callers run that layer under an external memory/
  CPU limit. Prod runtime is splashdown.service (plain uvicorn, NOT docker — the compose file is
  a dev artifact), so a compose MemoryLimit would be dead weight; recheck at scale.
- **Why.** Both write-ups' suggested fixes were verified against a topology the service does not
  run; the shipped rules are the deployment-aware generalisations, each with a live gate and a
  self-check assert.
