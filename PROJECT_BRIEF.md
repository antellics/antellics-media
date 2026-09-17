# Project Brief: Static Site Platform

A build-and-host platform for local small-business websites. One operator, target
200+ client sites, near-zero marginal cost per site.

This document is the starting spec. Read it fully before writing code.

---

## 1. Objective

Build a Python toolchain that takes a structured business record and produces a
deployed, validated, high-performance static website on a custom domain, with
minimal human intervention.

**Success criterion:** intake form submitted → site live on client's domain in
under one hour, zero manual steps in between.

---

## 2. Architecture

```
  ┌─────────────────────────────┐
  │  BUILD HOST (local desktop) │   24c / 128 GB / RTX 5090
  │                             │
  │  site.yaml ──► Jinja2 ──► HTML
  │  assets/   ──► encoders ──► AVIF/WebP/JPEG
  │  local LLM ──► copy, reports, log analysis
  │  QA gates  ──► Lighthouse CI, vnu, JSON-LD, NAP
  └──────────────┬──────────────┘
                 │ rsync over WireGuard
                 ▼
  ┌─────────────────────────────┐
  │  ORIGIN (Hetzner bare metal)│   Falkenstein or Helsinki
  │  Caddy, static files only   │
  │  + form handler (separate)  │
  └──────────────┬──────────────┘
                 │
                 ▼
         Cloudflare (DNS + edge cache + TLS to client)
```

**The origin is disposable.** It holds no secrets, no API keys, no model
weights, no canonical data. Everything is regenerable from git plus the build
host. A full rebuild onto fresh hardware must be one command.

---

## 3. Non-negotiable constraints

These are settled. Do not propose alternatives.

| Constraint | Rationale |
|---|---|
| Python 3.12+, async where I/O-bound (`httpx`, not `requests`) | Operator preference, existing tooling |
| No runtime code paths on the origin except the form handler | Attack surface |
| No database in the serving path | Origin must be stateless |
| No JS framework in output; vanilla JS only, minimal | Lighthouse budget, no build step for clients |
| Structured JSONL logs from every pipeline stage | Greppable, machine-analyzable |
| `StrictUndefined` in Jinja2 | Missing data must fail the build, not render blank |
| All LLM calls are build-time and local (Ollama / llama.cpp server) | Zero marginal cost, no request-time dependency |
| Content-addressed asset cache | Rebuild cost must scale with changes, not site count |
| Git is the source of truth for all tenant data | Recoverability |

---

## 4. Repository layout

```
platform/
├── pyproject.toml
├── src/platform/
│   ├── models.py          # pydantic models for site.yaml, intake, tenant
│   ├── render.py          # Jinja2 stage
│   ├── images.py          # encode pipeline, content-addressed cache
│   ├── llm.py             # local inference client (OpenAI-compatible endpoint)
│   ├── qa/
│   │   ├── lighthouse.py
│   │   ├── html.py        # vnu wrapper
│   │   ├── schema.py      # JSON-LD validation
│   │   └── nap.py         # NAP byte-match assertion
│   ├── deploy.py          # rsync + symlink swap, provider-agnostic
│   ├── dns.py             # Cloudflare API
│   ├── tenants.py         # SQLite tenant registry
│   └── cli.py             # entry point
├── templates/
│   ├── base.html.j2
│   ├── partials/
│   └── verticals/         # trades/, food/, professional/, retail/
├── prompts/               # versioned, one file per task
├── tenants/
│   └── <slug>/
│       ├── site.yaml
│       ├── intake.json
│       ├── dns.yaml
│       ├── provenance.jsonl
│       └── assets/
├── infra/
│   ├── playbook/          # origin provisioning (Ansible or shell, your call)
│   ├── Caddyfile.j2
│   └── systemd/
└── tests/
```

---

## 5. Data models

### `site.yaml` — single source of truth per client

Define as a pydantic model. Minimum fields:

```yaml
slug: acme-plumbing
vertical: trades
business:
  name: "Acme Plumbing"          # NAP-critical
  phone: "+1-508-555-0142"       # NAP-critical
  address:                        # NAP-critical
    street: "12 Main St"
    locality: "North Attleborough"
    region: "MA"
    postal_code: "02760"
    country: "US"
  hours:                          # → openingHoursSpecification
    mon: ["08:00", "17:00"]
  service_area: ["North Attleborough", "Plainville", "Wrentham"]
domains:
  primary: "acmeplumbing.com"
  aliases: ["www.acmeplumbing.com"]
content:
  headline: "..."                 # LLM-drafted, human-approvable
  services: [...]
  about: "..."
assets:
  logo: "assets/logo.svg"
  gallery: [...]
forms:
  recipients: ["owner@acmeplumbing.com"]
```

NAP fields are the contract. They must appear byte-identical in rendered HTML
and in JSON-LD. Enforce with an assertion, not a review step.

### `tenants.sqlite`

The only thing the origin reads at runtime, via the TLS `/check` endpoint.

```sql
CREATE TABLE tenants (
  slug        TEXT PRIMARY KEY,
  state       TEXT NOT NULL,   -- demo|pending|active|suspended
  plan        TEXT,
  billing_ref TEXT,
  created_at  TEXT NOT NULL
);
CREATE TABLE hostnames (
  hostname TEXT PRIMARY KEY,
  slug     TEXT NOT NULL REFERENCES tenants(slug),
  kind     TEXT NOT NULL       -- primary|alias|demo
);
```

---

## 6. Build pipeline

Model as a DAG keyed on content hashes. Skip unchanged work. Emit
`{slug, stage, status, duration_ms, hash}` JSONL per stage.

| Stage | Notes |
|---|---|
| **render** | Jinja2, `StrictUndefined`. Deterministic: same YAML → byte-identical HTML. Milliseconds. |
| **images** | Responsive widths (400/800/1200/1600) × AVIF + WebP + JPEG fallback. ~12 encodes per source. `ProcessPoolExecutor(max_workers=24)`. AVIF via `avifenc`/`cavif`, not Pillow. Cache key: `sha256(bytes + width + format + quality)`; output filename *is* the hash, enabling `Cache-Control: immutable`. |
| **qa:lighthouse** | Headless Chrome against local `caddy file-server`, never production. Median of 3 runs. Cap concurrency at 8–12 — Chrome instances skew each other's timings. Assert perf ≥ 0.95, a11y ≥ 0.95, SEO = 1.0. |
| **qa:html** | `vnu` jar, offline. |
| **qa:schema** | Parse `application/ld+json`, validate `LocalBusiness` shape. Malformed JSON-LD is silently dropped by Google — catch it here. |
| **qa:nap** | `assert rendered_nap == source_nap`. Non-negotiable. LLMs hallucinate phone numbers and normalize addresses. |
| **deploy** | `rsync` to `/srv/sites/<host>/releases/<sha>/`, then `ln -sfn` the `current` symlink. Rollback is one symlink swap. |

Any QA gate failing blocks deploy. No override flag.

---

## 7. CLI surface

```
platform demo <prospect.json>       # prospect → live demo subdomain
platform intake <slug> <form.json>  # intake → site.yaml draft
platform build <slug> [--all]       # render + images + QA
platform deploy <slug> [--target H] # push to origin; target is a hostname
platform dns apply <slug>           # reconcile Cloudflare zone to dns.yaml
platform activate <slug>            # flip tenant state, enable real-domain TLS
platform report <slug> --month YYYY-MM
platform rebuild-origin <host>      # bare host → serving, one command
```

`--target` and `rebuild-origin` are what make the origin disposable. Build them
from day one; retrofitting provider-agnosticism is painful.

---

## 8. Origin provisioning

Reproducible from the playbook. Nothing configured by hand.

- Caddy under a dedicated uid. systemd unit with `ProtectSystem=strict`,
  `NoNewPrivileges=yes`, `PrivateTmp=yes`, `SystemCallFilter=@system-service`,
  `CapabilityBoundingSet=CAP_NET_BIND_SERVICE`.
- Content dirs bind-mounted read-only into the service namespace.
- On-demand TLS gated by `ask http://127.0.0.1:9000/check` — a small FastAPI
  service reading `tenants.sqlite`. 200 for active tenants and demo hostnames,
  404 otherwise. Without the gate this is a cert-exhaustion DoS.
- Wildcard cert for `*.demo.<yourdomain>`.
- SSH bound to a WireGuard interface. Nothing on public :22.
- Default-deny egress. Port 25 blocked permanently.
- Security headers: HSTS w/ preload, CSP, `X-Content-Type-Options`,
  `Referrer-Policy`, `-Server`.
- `unattended-upgrades` on.

**Form handler** is a separate systemd unit under a separate uid: Turnstile
gate, per-tenant HMAC baked into the build, rate limits per IP and per tenant,
outbound mail via Postmark API only.

---

## 9. Local LLM usage

All build-time. Hit a local OpenAI-compatible endpoint. Never at request time.

| Task | Notes |
|---|---|
| Copy drafting | Input is `intake.json` + mined reviews. Output is `site.yaml` content fields — structured, never raw HTML. |
| Review mining | *Extraction*, not summarization. What customers repeatedly praise becomes the headline. |
| Monthly report narrative | GBP Insights + Search Console + Plausible → templated prose → PDF. |
| Log analysis | Weekly access-log digest. "What changed?" |

Prompts live in `prompts/`, versioned, one file per task. Log prompt version
alongside output so a regression is traceable.

---

## 10. Milestones

Build in this order. Each must be independently demonstrable.

1. **Render + deploy.** Hand-written `site.yaml` → HTML → live on a demo
   subdomain with working TLS. Proves the whole spine.
2. **Origin playbook.** `rebuild-origin` takes bare Ubuntu → serving. Verify by
   destroying and rebuilding.
3. **Image pipeline.** Content-addressed cache. Benchmark cold vs. warm build.
4. **QA gates.** All four, wired as blockers.
5. **Tenant registry + `/check`.** Real custom domains via on-demand TLS.
6. **DNS automation.** Cloudflare zone reconciliation from `dns.yaml`.
7. **Demo generator.** Prospect record → live demo, unattended, batchable.
8. **Form handler.**
9. **Reporting.**

---

## 11. Guardrails

- Do not add a CMS, admin panel, or client login. Not in scope.
- Do not add a database to the serving path.
- Do not introduce React/Vue/Svelte into the output.
- Do not call any LLM API at request time.
- Do not write a QA-gate bypass flag.
- Do not cache or persist Google Places photo content — the Places API terms
  prohibit it. Demo imagery must be licensed stock; real photos get shot at
  signing.
- Record image license provenance in `provenance.jsonl` at ingest. A DMCA
  notice carries a 24-hour response window; you want a receipt, not a search.

---

## 12. Open questions

Flag these rather than guessing:

- Orchestration: hand-rolled DAG vs. `doit` vs. Makefile? Prefer the smallest
  thing that handles content-hash invalidation.
- Ansible vs. plain shell for the playbook.
- Which local model for copy drafting — needs evaluation against real intake
  data, not vibes.
- Registrar choice (API quality is the deciding factor).

---

## 13. Compliance note

Contact form submissions will contain Massachusetts residents' personal
information, processed on behalf of client businesses. 201 CMR 17.00 requires a
written information security program, and it reaches service providers. The WISP
must exist before the first form submission. Keep the form handler on
US infrastructure so the data-transfer question never arises.
