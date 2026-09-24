# Review: PROJECT_BRIEF.md

**Date:** 2026-09-17
**Scope:** Spec review only. Repo is greenfield — `PROJECT_BRIEF.md` plus a bare
`pyproject.toml`. No implementation exists yet.
**Purpose:** Surface contradictions, gaps, and failure modes in the brief *before*
code encodes them, and propose a concrete resolution for each.

---

## How to read this

Each item is **Concern → Impact → Proposed solution**. Severity:

| Level | Meaning |
|---|---|
| **BLOCKING** | Two settled decisions contradict each other. Whoever writes code first resolves it silently and probably badly. Resolve before implementation. |
| **HIGH** | Real operational, financial, or legal consequence at 200-site scale. |
| **MEDIUM** | Will cause rework or erode trust in the pipeline. |
| **LOW** | Correctness nit. Cheap now, annoying later. |

Items are numbered `#1`–`#28` and stable — reference them by number.

---

## Decision register

Everything else in this document is downstream of these four. They are **not**
in the brief's §12 open questions, and unlike those four (DAG tool, Ansible vs.
shell, which local model, registrar) they are expensive to reverse.

| # | Decision | Blocks | Status |
|---|---|---|---|
| D1 | Cloudflare full-zone vs. Cloudflare for SaaS — and therefore whether on-demand TLS exists at all | `dns.py`, `deploy.py`, `/check`, activation flow, milestones 2/5/6 | **OPEN** |
| D2 | Who owns client domains and DNS; what the offboarding path is | Onboarding contract, `dns.py`, churn handling | **OPEN** |
| D3 | Performance gating: Lighthouse score vs. deterministic budgets | `qa/lighthouse.py`, whether the pipeline is trustworthy | **OPEN** |
| D4 | Secret storage and build-host disaster recovery | Everything. The single point of failure currently has no plan. | **OPEN** |

The brief's own §12 questions are all reversible in an afternoon. These are not.

---

# A. Contradictions to resolve before writing code

## #1 — Two TLS terminators, neither declared authoritative
**Severity:** BLOCKING · **Refs:** §2, §8

**Concern.** §2 puts Cloudflare in front doing "TLS to client." §8 has Caddy
doing on-demand TLS gated by `/check`. These are alternative designs, not
layers, and the brief never says which one is real.

**Impact.** If Cloudflare proxies (orange cloud), it terminates client TLS and
the origin certificate only covers the CF↔origin hop — where on-demand issuance
is pointless and the correct answer is a Cloudflare Origin CA cert with Full
(strict) plus authenticated origin pulls. If the origin is DNS-only (grey
cloud), on-demand TLS works but you lose the edge cache, DDoS absorption, and
the entire reason Cloudflare is in the diagram.

**Proposed solution.** For 200 client-*owned* domains the shape that actually
fits is **Cloudflare for SaaS custom hostnames**: clients CNAME to you, you
provision hostnames via API, Cloudflare handles client-facing certs. Note this
is per-hostname pricing and a *different* API surface than "reconcile a zone
from `dns.yaml`" — model it in `dns.py` from day one rather than retrofitting.

Whichever way D1 lands, write it into §2 explicitly: who terminates client TLS,
what cert the origin presents, and whether the origin is proxied.

---

## #2 — On-demand TLS is probably unnecessary
**Severity:** BLOCKING · **Refs:** §5, §8

**Concern.** On-demand TLS exists for hostnames you *cannot know in advance*.
You know every hostname at build time — that is precisely what `tenants.sqlite`
is.

**Impact.** The current design carries a FastAPI service, a SQLite replica on
the origin, an unspecified DB sync mechanism, and an acknowledged
cert-exhaustion DoS vector — all to answer a question the build host already
knows the answer to.

**Proposed solution.** Generate explicit Caddy site blocks from the tenant
registry at deploy time and reload. This deletes, in one move:

- the `/check` FastAPI service (and with it, exception #4 below)
- `tenants.sqlite` on the origin, and the sync/atomicity problem
- the cert-exhaustion attack surface

**Tradeoff to accept consciously.** You lose graceful handling of the window
where a tenant is activated but DNS is not yet pointed — issuance fails and
Caddy retries in the background. Handle that by gating `platform activate` on a
DNS pre-check, which is worth doing regardless (see #22).

If D1 lands on Cloudflare for SaaS, this becomes moot in a different way: CF
issues the client-facing cert and the origin serves a single wildcard/Origin CA
cert.

---

## #3 — "No runtime code paths except the form handler" already has two exceptions
**Severity:** MEDIUM · **Refs:** §3, §5, §8

**Concern.** §3 lists one exception. §8 introduces a second (the `/check`
FastAPI service). §5 reinforces it ("the only thing the origin reads at
runtime").

**Impact.** A non-negotiable constraint violated inside the same document trains
everyone — human and agent — to treat the constraint list as aspirational.

**Proposed solution.** Eliminate the service per #2, or amend §3 to name both
exceptions explicitly. Do not leave it ambiguous.

---

## #4 — The NAP byte-identity assertion cannot work as written
**Severity:** BLOCKING · **Refs:** §5, §6

**Concern.** Source is `+1-508-555-0142`. Rendered HTML wants `(508) 555-0142`
for humans, `tel:+15085550142` in the `href`, and JSON-LD wants E.164
`+15085550142`. That is three distinct byte sequences from one source field.
`assert rendered_nap == source_nap` either fails on day one or forces
user-hostile output. Same problem for address: JSON-LD `PostalAddress` fields
vs. a comma-joined display string.

**Impact.** The gate the brief calls "non-negotiable" is the one most likely to
be quietly loosened into uselessness.

**Proposed solution — guarantee by construction, then assert.** The intent is
right; the mechanism is doing work that construction should do.

1. **NAP never passes through the LLM.** Templates read NAP fields directly from
   the pydantic model. The LLM receives and returns only non-NAP content fields.
   This removes the hallucination vector entirely rather than detecting it after
   the fact.
2. **Validate LLM output** rejects any phone-shaped or address-shaped string,
   as defense in depth.
3. **Canonicalize, don't string-compare.** A single `format_phone(source)` /
   `format_address(source)` module emits the known-good set of display,
   `tel:`, and JSON-LD variants. The gate asserts every NAP occurrence in
   rendered HTML and JSON-LD is a *member of that set*.

Net: a stronger guarantee than a post-hoc byte compare, and an assertion that
can actually pass.

---

## #5 — `src/platform/` shadows a Python standard library module
**Severity:** HIGH · **Refs:** §4

**Concern.** `import platform` is stdlib. The brief's repository layout names
the package `platform`.

**Impact.** Breaks in editable installs, test collection, and anything importing
stdlib `platform` — which includes parts of `setuptools` and assorted
diagnostic code. Symptoms are confusing and appear far from the cause.

**Proposed solution.** Rename now. The repo is already `antellics-media`;
`src/antellics/` or `src/sitegen/` costs nothing today. Update §4 and the CLI
entry point name accordingly.

---

# B. Gaps with operational or legal consequence

## #6 — Taking over client DNS can break their email
**Severity:** HIGH · **Refs:** §7, §4 (`dns.yaml`)

**Concern.** `platform dns apply` reconciling a Cloudflare zone from `dns.yaml`
implies you hold the full zone. For any client on Google Workspace or Microsoft
365, a reconciler treating `dns.yaml` as complete truth will drop MX, SPF,
DKIM, and DMARC records.

**Impact.** Silently killing a small business's email is close to
business-ending for them and reputation-ending for you — and you would be doing
it unattended, 200 times.

**Proposed solution.** `dns.yaml` needs an explicit **ownership model**:

- On zone adoption, **import** all existing records first and snapshot them into
  the tenant directory (git-tracked, so drift is visible).
- Each record is classified `managed` (you own it, reconcile freely) or
  `carried` (preserved untouched, never deleted).
- Destructive operations require an explicit diff approval — `dns apply` prints
  the plan, `dns apply --confirm <plan-hash>` executes it.
- Hard rule in code: **never delete a record class you do not own.** MX, TXT,
  and anything unrecognized default to `carried`.

---

## #7 — No offboarding or portability story
**Severity:** HIGH · **Refs:** §7, §11

**Concern.** Some fraction of 200 clients will leave. If you hold their domain
registration and DNS, and their content exists only in your private git repo,
churn becomes a hostage negotiation.

**Impact.** Legal and reputational exposure, concentrated at exactly the moment
a relationship has already soured.

**Proposed solution.**
- **Clients register and own their domains**; they grant you delegated access.
  Resolve as part of D2 and write it into the onboarding contract.
- Add `platform offboard <slug>`: exports rendered site + source content in a
  portable form, emits a DNS handoff record, and flips tenant state.
- Build it while you have zero clients and it is nearly free.

---

## #8 — Consumer PI and git are on a collision course
**Severity:** HIGH · **Refs:** §3, §13

**Concern.** §3 makes git the source of truth for all tenant data. §13 correctly
flags 201 CMR 17.00. Nothing states the invariant that lets both survive.

**Impact.** Anything committed is effectively permanent and cannot be honored
against a deletion request. The repo currently has **no `.gitignore`** — which
is exactly how the first secret or PII file gets committed.

**Proposed solution.** State and enforce the invariant explicitly in §13:

> **No consumer personal information ever enters the repository or the build
> host.** Form submissions relay through the handler to Postmark and are never
> persisted; submission bodies are never logged.

Enforcement:
- Add a `.gitignore` before the first real commit.
- Pre-commit hook scanning for phone/email/secret patterns in `tenants/`.
- Form handler logs metadata only (timestamp, tenant, disposition, rate-limit
  state) — never body content.

---

## #9 — §13 is missing most of what a WISP requires
**Severity:** HIGH · **Refs:** §13

**Concern.** The brief correctly notes the WISP must exist before the first form
submission, but stops there.

**Impact.** A WISP that exists but omits required elements provides no
protection and is discoverable in exactly the scenario where it matters.

**Proposed solution.** The WISP must additionally cover:

- **Retention and minimization** for the form handler — ideally "in-flight
  only, never persisted," which dramatically shrinks compliance scope and
  should be an explicit design goal, not an accident.
- **Breach runbook.** M.G.L. c. 93H requires notice to the Attorney General and
  OCABR. Write the runbook before you need it.
- **Processor terms** between you and each client — you are a service provider
  handling their customers' PI; 201 CMR 17.00 reaches service providers.
- **Subprocessor inventory.** Postmark retains message content for a period by
  default (verify current retention settings) — that is a subprocessor holding
  PI and belongs in the inventory with its own due-diligence record.
- **Designated responsible person, annual review, incident log.**

Add the WISP to §10 as a numbered milestone gating the form handler, not as a
closing note.

---

## #10 — Secrets and build-host DR are entirely unspecified
**Severity:** HIGH · **Refs:** §2, §3

**Concern.** "Everything is regenerable from git plus the build host" — nothing
says what regenerates the build host. It holds Cloudflare tokens, Postmark
keys, WireGuard keys, registrar credentials, and per-tenant HMACs.

**Impact.** The most valuable machine in the architecture has no backup story,
while the document's stated design goal is disposability.

**Proposed solution.**
- **Derive per-tenant HMACs**: `HMAC(master_secret, slug)`. One secret to
  protect, not 200, and rotation is a single rebuild.
- **Encrypted secrets in the repo** via age/sops. DR becomes "clone + one
  private key," which is a story you can actually test.
- Add `platform rebuild-buildhost` or at minimum a documented DR runbook, and
  **test it once** — the brief already applies this standard to the origin
  (§10.2, "verify by destroying and rebuilding"). Hold the build host to it too.

---

## #11 — No fleet observability
**Severity:** HIGH · **Refs:** §2, §8, §10

**Concern.** Nothing monitors the fleet. At 200 domains, an expired cert, a
client who moved their nameservers, or an origin outage is discovered by angry
phone call.

**Impact.** Support load scales with failures you did not detect, which
undermines the "near-zero marginal cost" premise more than build cost does.

**Proposed solution.** A monitoring stage, run from the build host (not the
origin), covering:

- per-hostname uptime and HTTP status
- certificate expiry
- **DNS drift** — live zone vs. `dns.yaml`, catches clients who changed
  nameservers or a registrar that reset records
- **release drift** — deployed SHA vs. expected SHA per host
- add `platform status [--all]` to the CLI surface (§7)

**Also:** one origin is one failure domain for 200 paying clients. `--target`
exists precisely to enable a second origin — promote "deploy to two origins"
from latent capability to an explicit near-term goal.

---

## #12 — Demo sites of businesses that did not ask for them
**Severity:** HIGH · **Refs:** §9, §11, §10.7

**Concern.** The demo generator builds sites using another business's name,
address, and reviews, and publishes them on a public subdomain.

**Impact.** Brand confusion, duplicate-content competition against the
prospect's real site, trademark exposure, and indefinite accumulation of
unsolicited material bearing other companies' names.

**Proposed solution.**
- **`noindex` plus `robots.txt` disallow across the entire demo wildcard.**
  Non-optional; make it a QA gate on demo builds.
- Visible "unofficial demo" labeling on every demo page.
- **Auto-expiry on demo tenants** — add `expires_at` to the registry and a
  reaper. Prevents indefinite accumulation and bounds the exposure window.

**Related:** §9's "review mining" may conflict with §11's own caution. Places
API terms restrict copying and storing review *content*, not only photos —
verify against current terms before building that stage, since mined review text
is meant to become the headline.

---

## #13 — No URL migration / redirect map
**Severity:** HIGH · **Refs:** §5, §6

**Concern.** The product's value proposition is local SEO. For any client with
an existing website, replacing it without 301s from the old URLs discards
whatever ranking they had.

**Impact.** You can measurably harm the exact metric you are selling — on the
clients who have the most to lose, because they are the ones with an existing
site.

**Proposed solution.**
- Add a `redirects:` section to `site.yaml` (old path → new path).
- Capture the old URL inventory at intake (crawl their existing site, or
  Search Console export once verified).
- Render to Caddy redirect directives at deploy.
- Add a `qa:redirects` check asserting every catalogued old URL resolves 301 to
  a live 200.

---

# C. Where the gates will fail you

## #14 — Lighthouse perf ≥ 0.95 as an unbypassable blocker will false-fail
**Severity:** BLOCKING (D3) · **Refs:** §6

**Concern.** The brief already acknowledges Chrome instances skew each other's
timings. Median-of-3 at 8–12 concurrency still leaves several points of
variance. A localhost score against `caddy file-server` also does not predict
production behind Cloudflare — it mostly measures your own HTML/CSS/image
discipline.

**Impact.** A hard gate on a noisy measurement will block legitimate client
updates. That is how gates get disabled.

**Proposed solution — split the gates by determinism.**

| Gate | Nature | Use as |
|---|---|---|
| `qa:html` (vnu) | Deterministic | **Hard blocker** |
| `qa:schema` (JSON-LD) | Deterministic | **Hard blocker** |
| `qa:nap` | Deterministic (post #4) | **Hard blocker** |
| Lighthouse **accessibility** | Near-deterministic (axe-based) | **Hard blocker** ✅ keep at 0.95 |
| Lighthouse **SEO** | Near-deterministic static checks | **Hard blocker** ✅ keep at 1.0 |
| Lighthouse **performance** | Sampled measurement | **Trend/warn only** |

Replace the perf blocker with **deterministic budgets**, which catch the same
regressions and are byte-exact and reproducible:

- total transferred bytes, per page and per site
- request count
- AVIF + WebP present at every declared width
- explicit `width`/`height` on every `<img>` (CLS)
- LCP element preloaded
- zero render-blocking resources

Keep Lighthouse perf running and tracked over time — it is genuinely useful as
a signal. It is just not a gate.

---

## #15 — "No override flag" needs a legitimate escape hatch
**Severity:** MEDIUM · **Refs:** §6, §11

**Concern.** A hard rule with no legitimate exit gets bypassed by disabling the
check entirely.

**Impact.** The failure mode being avoided is a lazy `--force`. The failure mode
being created is a client site that cannot ship because of measurement noise —
which eventually produces a commit deleting the gate.

**Proposed solution.** Keep the intent, change the mechanism: **no CLI flag**,
but allow a committed per-tenant waiver file:

```yaml
# tenants/<slug>/waivers.yaml
- gate: qa:lighthouse:perf
  reason: "Client-supplied hero video, approved 2026-09-17"
  expires: 2026-12-17
```

Auditable, reviewable in a diff, and expires on its own. A build fails if a
waiver is past its expiry.

---

## #16 — Nobody tests the QA gates
**Severity:** MEDIUM · **Refs:** §4 (`tests/`), §6

**Concern.** `tests/` appears in the layout with no strategy attached. Four of
nine milestones depend on the gates.

**Impact.** A gate that silently passes is worse than no gate — it manufactures
false confidence at exactly the point the design says to rely on it.

**Proposed solution.** Every gate ships with a **known-bad fixture it must
fail**, run in CI:

- `qa:nap` → fixture with a transposed phone digit
- `qa:schema` → fixture with malformed JSON-LD
- `qa:html` → fixture with an unclosed tag
- `qa:redirects` → fixture with a missing 301

Plus golden-file tests for render determinism (#17) and a canonical fixture
tenant used across the suite.

---

# D. Correctness and completeness

## #17 — Determinism needs active guarding
**Severity:** MEDIUM · **Refs:** §6

**Concern.** "Same YAML → byte-identical HTML" is asserted, not enforced.

**Proposed solution.** Enforce explicitly:
- No timestamps in output. A `{{ now().year }}` footer breaks byte-identity and
  triggers a full 200-site rebuild every January 1 — decide deliberately.
- Sorted iteration over any mapping.
- Pinned Jinja2 and Python versions.
- Locale-independent formatting throughout.
- Golden-file test: render fixture twice in separate processes, assert
  identical bytes.

---

## #18 — Sparse `hours` fights `StrictUndefined`
**Severity:** MEDIUM · **Refs:** §5, §3

**Concern.** The `site.yaml` example defines only `mon`. Any template touching
`hours.tue` explodes for a business closed Tuesdays — and `StrictUndefined`
means it explodes at build time, which is the stated intent but the wrong
trigger.

**Proposed solution.** Model all seven days explicitly in the pydantic model,
with a closed marker rather than absence:

```yaml
hours:
  mon: ["08:00", "17:00"]
  tue: closed
  # ... all seven required
```

Validation rejects a partial week. Maps cleanly to
`openingHoursSpecification`.

---

## #19 — Image cache key is incomplete
**Severity:** MEDIUM · **Refs:** §6

**Concern.** `sha256(bytes + width + format + quality)` omits the encoder.

**Impact.** An `avifenc` upgrade or settings change silently serves stale
outputs forever — the cache never invalidates because the key cannot see the
change.

**Proposed solution.** Include **encoder identity and version** plus the full
settings tuple in the key. Bonus: makes a deliberate encoder upgrade a
one-line, fully-invalidating change.

---

## #20 — EXIF is not stripped
**Severity:** HIGH · **Refs:** §6, §11

**Concern.** Client-supplied job-site photos carry GPS coordinates — frequently
a customer's home address — plus device and timestamp metadata.

**Impact.** Publishing a customer's home coordinates on a public website. A
privacy leak with a compliance flavor, on behalf of a client who has no idea.

**Proposed solution.** Strip all metadata at ingest, before hashing. Record the
strip in `provenance.jsonl` alongside license provenance. Add a QA assertion
that no output image carries EXIF.

---

## #21 — CSP vs. inlined critical CSS is a real tension
**Severity:** MEDIUM · **Refs:** §8, §6

**Concern.** §8 requires a CSP. A strict `style-src 'self'` /
`script-src 'self'` without `'unsafe-inline'` conflicts with inlining critical
CSS for LCP, which the perf budget wants.

**Proposed solution.** Compute `'sha256-...'` hashes for each inline block at
build time and emit them into the CSP header (rendered into the Caddy config
per site). The build pipeline is well positioned to do this — but it has to be
*designed in*, because retrofitting means touching the render stage, the header
generation, and the deploy config simultaneously.

Write the exact intended CSP into §8 rather than leaving it as "CSP."

---

## #22 — Rollback ignores the edge cache, and asset GC is unspecified
**Severity:** MEDIUM · **Refs:** §6

**Concern.** "Rollback is one symlink swap" is true for files on disk.
Cloudflare still holds the old HTML.

**Proposed solution.**
- **Cache policy, explicitly:** hashed assets `Cache-Control: immutable,
  max-age=31536000`; HTML `max-age=0, must-revalidate` with ETag revalidation
  at the edge. Optionally purge-by-URL on deploy.
- **Decide where images live.** In the release dir → duplication across
  releases. In a shared content-addressed pool → rollback is clean, but GC must
  never delete an asset referenced by *any retained release*. Recommend the
  shared pool with refcounting across retained releases.
- Add `platform rollback <slug> [--to <sha>]` to §7 — rollback is a stated
  design goal with no command.

---

## #23 — Registry schema gaps
**Severity:** MEDIUM · **Refs:** §5

**Concern.** The `tenants` / `hostnames` schema is missing fields the rest of
the design depends on.

**Proposed solution.** Add:
- `updated_at` on both tables
- `current_release_sha` — required for rollback (#22) and drift detection (#11)
- `expires_at` on demo tenants (#12)
- `CHECK (state IN ('demo','pending','active','suspended'))` — currently just
  a string
- index on `hostnames(slug)`
- **Resolve `billing_ref`.** It implies billing exists, but billing appears
  nowhere in the CLI or milestones. Either scope it or drop the column.

---

## #24 — Missing CLI verbs
**Severity:** MEDIUM · **Refs:** §7

**Concern.** Several stated design goals have no command.

**Proposed solution.** Add to §7:
- `platform rollback <slug> [--to <sha>]` — stated goal, no command (#22)
- `platform approve <slug>` — §5 calls headlines "human-approvable" but there is
  no approval verb and no approval state anywhere in the data model
- `platform offboard <slug>` (#7)
- `platform status [--all]` (#11)

---

## #25 — Undefined input schemas and unplaced dependencies
**Severity:** MEDIUM · **Refs:** §7, §9

**Concern.**
- `prospect.json` and `form.json` are CLI inputs with no pydantic models —
  only `site.yaml` is modeled.
- §9's reporting depends on GBP Insights, Search Console, and Plausible. None
  appear in the architecture diagram, `infra/`, or the milestone list.
- GBP API access requires each client to grant manager rights on their Google
  Business Profile — a real onboarding step with real friction, unmentioned.

**Proposed solution.** Model all three inputs in `models.py`. Add the analytics
stack as an explicit decision (self-hosted Plausible is another service to run —
it is not in the serving path, so it violates nothing, but it needs a home) and
add the GBP access grant to the onboarding checklist.

---

## #26 — Missing output files and link checking
**Severity:** LOW · **Refs:** §6

**Proposed solution.** Add to the render stage: `sitemap.xml`, `robots.txt`,
404 page, favicon/app icons. Add a `qa:links` gate for broken internal links and
dead anchors — cheap, deterministic, and catches a whole class of template bugs.

---

## #27 — Security detail refinements
**Severity:** MEDIUM · **Refs:** §8

**Concern / proposed solution:**

- **Do not HSTS-preload client domains.** Preload is domain-wide, applies to
  subdomains the client may use for other services, and is slow and painful to
  reverse. Preloading someone else's domain is not your decision to make
  unilaterally. HSTS *without* preload is correct here. Amend §8.
- **Default-deny egress needs a concrete allowlist.** ACME endpoints are
  CDN-backed with unstable IPs; combined with Postmark and the Cloudflare API,
  a pure IP allowlist is impractical. Plan for a DNS-based allowlist or an
  egress proxy, or the rule gets relaxed to nothing on first contact.
- **Harden further:** `ProtectHome`, `RestrictAddressFamilies`,
  `MemoryDenyWriteExecute`, narrow `ReadWritePaths`.
- **rsync user ≠ caddy user.** The rsync target sits outside the service
  namespace; release directories become immutable after the symlink swap.
- **The WireGuard/deploy key on the build host is the crown jewel.** Folds into
  D4 (#10).

---

## #28 — Repo-state nits
**Severity:** LOW

- **No `.gitignore`** — add before the first real commit (#8).
- `pyproject.toml` declares `requires-python = ">=3.14"`; the brief says
  3.12+. Reconcile. 3.14 is fine but pins you to the newest runtime, where some
  dependencies may lag on wheels — confirm that is deliberate.
- `description = "Add your description here"` is still the uv default.

---

# E. Proposed milestone re-ordering

Two problems with §10 as written.

**1 before 2 is backwards.** You would hand-build an origin, then write a
playbook that encodes your hand-built drift. Either swap them, or declare
milestone 1's origin explicitly throwaway.

**The form handler at #8 is far too late.** Milestone 5 puts real clients on
real domains — with a non-functional contact form. For a local trades business
the contact form *is* the conversion path and the reason they are paying you.
And since §13 requires the WISP before the first submission, the WISP is a
milestone, not a footnote.

**Proposed order:**

| # | Milestone | Change |
|---|---|---|
| 1 | Origin playbook — bare Ubuntu → serving, verified by destroy/rebuild | ⬆ was 2 |
| 2 | Render + deploy — hand-written `site.yaml` → live demo subdomain with TLS | ⬇ was 1 |
| 3 | Image pipeline — content-addressed cache, cold vs. warm benchmark | — |
| 4 | QA gates — per #14's blocker/trend split, all wired | — |
| 5 | Tenant registry + hostname provisioning (shape depends on D1) | — |
| 6 | DNS automation — with the ownership model from #6 | — |
| 7 | **WISP written and in force** | ⬆ **new** (was a §13 note) |
| 8 | **Form handler** | ⬆ was 8, now gates first activation |
| 9 | **First real client activation** | ⬆ **new explicit gate** |
| 10 | Demo generator — with noindex/robots/expiry from #12 | ⬇ was 7 |
| 11 | Reporting | — |

The key move: **nothing activates on a real client domain until the form handler
and the WISP both exist.**

---

# F. Quick-fix checklist

Items that are cheap now and expensive later, independent of the four decisions:

- [ ] Rename `src/platform/` → `src/antellics/` (#5)
- [ ] Add `.gitignore` (#8, #28)
- [ ] Fix `pyproject.toml` description and reconcile Python version (#28)
- [ ] Add encoder version to the image cache key (#19)
- [ ] Specify EXIF stripping at ingest (#20)
- [ ] Make `hours` a complete seven-day model (#18)
- [ ] Amend §8: HSTS without preload on client domains (#27)
- [ ] Amend §3: name both runtime exceptions, or remove one (#3)
- [ ] Add `rollback`, `approve`, `offboard`, `status` to §7 (#24)
- [ ] Add `current_release_sha`, `updated_at`, `expires_at`, state CHECK to the
      registry schema (#23)

---

# G. Summary

The brief is strong — it states constraints *with rationale*, which is what
makes it reviewable rather than merely followable. The problems cluster in three
places:

1. **The TLS/DNS/Cloudflare boundary** (#1, #2, #3, #6, #7) is genuinely
   underspecified and everything else depends on it. This is D1 and D2.
2. **Two gates are stated as absolute but measured or specified in ways that
   cannot hold** — Lighthouse perf (#14) and NAP byte-identity (#4). Both get
   quietly disabled unless the mechanism changes.
3. **The operational surface at 200 clients is thinner than the build surface** —
   no monitoring (#11), no offboarding (#7), no build-host DR (#10), no redirect
   migration (#13). The build pipeline is designed to scale; the operation
   around it is not yet.

None of this is structural. The architecture is sound and the disposable-origin
instinct is right. These are resolvable now, on paper, for far less than they
cost later.
