---
title: "AutoRelay — Project Plan"
subtitle: "Authorized Active Directory Assessment Tool"
author: "Internal Working Document"
date: "Draft v1.0"
---

\newpage

# 0. Framing

**What we're building:** an authorized Active Directory assessment tool that
(a) checks whether an NTLM relay chain is viable against a target, (b) runs
the chain if the operator confirms, (c) logs everything, and (d) produces a
client-facing remediation report.

**What we're not building:** automatic DCSync → krbtgt forging. That step
stays manual and out of the auto-chain. We will document how to do it by
hand, but the tool will not do it.

**Design principle that governs every decision below:** *check loudly, fire
deliberately, report thoroughly.*

\newpage

# 1. Scope and Boundaries

## In scope

- Preflight vulnerability assessment (SMB signing, LDAP signing/channel
  binding, coercion vector availability)
- NTLM relay orchestration (wrapping Impacket / ntlmrelayx)
- Pluggable coercion modules (PetitPotam, DFSCoerce, PrinterBug, Responder)
- Opt-in post-exploitation (LDAP dump, SMB exec, secretsdump when the
  identity is privileged)
- Structured logging and remediation reporting

## Out of scope (deliberate)

- Auto-forging Golden Tickets
- Any action against hosts not in the operator-supplied target list
- Silent or background operation — every fire is either interactive or
  explicitly `--yes`

## Hard rules baked into the code

- Preflight must pass before any coercion fires.
- Coercion only against the single `--target` supplied.
- Relay only to hosts in `--relay-targets`.
- Post-exploitation requires an explicit `--post` flag; the default is
  "report only."

\newpage

# 2. Architecture

```text
autorelay/
├── autorelay.py              # CLI entry, orchestrator
├── config.yaml               # defaults, timeouts, paths
├── preflight/
│   ├── __init__.py           # runs all checks, returns PreflightReport
│   ├── smb_signing.py        # SMB signing on target + relay candidates
│   ├── ldap_signing.py       # LDAP signing / channel binding
│   ├── coercion_probe.py     # which coercion vectors respond
│   ├── reachability.py       # port-level reachability
│   └── report.py             # PreflightReport dataclass + renderer
├── coercion/
│   ├── base.py               # CoercionModule ABC
│   ├── petitpotam.py
│   ├── dfscoerce.py
│   ├── printerbug.py
│   └── responder.py           # passive LLMNR/NBT-NS/mDNS
├── relay/
│   ├── server.py             # wraps ntlmrelayx, event queue
│   └── session.py            # session detection + metadata
├── post/
│   ├── ldap_dump.py
│   ├── smb_exec.py
│   └── secretsdump.py
├── report/
│   ├── findings.py           # machine-readable results
│   ├── markdown.py           # client-facing report
│   └── remediation.py        # remediation table generator
├── util/
│   ├── log.py                # timestamped structured logging
│   └── net.py                # small networking helpers
└── tests/
    ├── test_preflight.py
    ├── test_coercion_parsers.py
    └── fixtures/             # captured nmap/ntlmrelayx output
```

**Dependency choice:** use Impacket as a library wherever practical. Shell out only to `ntlmrelayx.py` (its CLI is well-tested and its internals shift between releases). Never reimplement NTLM.

\newpage

# 3. Stages

Each stage is independently useful and independently shippable. We do not
move to the next until the current one works in the lab.

## Stage 1 — Preflight (build first, ship first)

**Goal:** given a target and a relay-target list, produce a clear report of
whether the chain is viable and why.

**Components:**

- `smb_signing.py` — negotiate SMB2, read signing-required flag on target
  and each relay candidate
- `ldap_signing.py` — query rootDSE, check LDAP signing and channel binding
  policy
- `coercion_probe.py` — bind to EFSRPC / DFSNM / Spooler RPC endpoints,
  report which respond
- `reachability.py` — TCP connect checks on 445, 389, 135, 139
- `PreflightReport` — dataclass, JSON-serializable, human-readable renderer

**Exit criteria:**

- Runs against GOAD (or your lab) and correctly identifies signing state on
  at least three hosts
- Correctly identifies at least one available coercion vector
- Produces clean JSON and Markdown output
- Never sends a coercion RPC (verified by packet capture)

**Deliverable:** `autorelay preflight --target X --relay-targets Y`

## Stage 2 — Reporting and Remediation

**Goal:** even with only preflight, the tool produces something a client can
act on.

**Components:**

- `remediation.py` — maps each finding to a concrete fix (GPO setting,
  registry key, patch)
- `markdown.py` — renders the report with finding / impact / remediation
  columns
- Severity model (blocker / warning / info) so the report prioritizes

**Exit criteria:**

- Running preflight against the lab produces a report where every finding
  has a remediation
- Report is readable by a non-operator (test it on a colleague)

**Deliverable:** `autorelay preflight ... --report out.md`

## Stage 3 — Relay Wrapper

**Goal:** cleanly start, monitor, and stop an ntlmrelayx instance, and
detect when a session lands.

**Components:**

- `relay/server.py` — subprocess lifecycle, output pump, event queue
- `relay/session.py` — parse ntlmrelayx output to detect successful relay,
  extract identity and target
- Graceful shutdown on Ctrl-C and on `--stop-after-first`

**Exit criteria:**

- Can start ntlmrelayx against lab relay targets, capture a relayed auth
  from a manual trigger, and report the session
- All output timestamped and captured to a log file
- Clean shutdown with no orphan processes

**Deliverable:** `autorelay relay --targets Y --listener IP`

## Stage 4 — First Coercion Module (PetitPotam)

**Goal:** prove the end-to-end chain with one coercion vector.

**Components:**

- `coercion/base.py` — `CoercionModule` ABC (`available()`, `trigger()`)
- `coercion/petitpotam.py` — EFSRPC-based coercion via Impacket
- Orchestrator wiring: preflight → start relay → trigger coercion → wait
  for session → report

**Exit criteria:**

- Full chain works in the lab: preflight passes, relay session lands,
  report includes the session
- Failure modes are informative: if coercion does not land, the tool says
  which vector failed and why
- Still no auto post-exploitation

**Deliverable:** `autorelay run --target X --relay-targets Y --listener IP`

## Stage 5 — Additional Coercion Modules

**Goal:** pluggability, and coverage for environments where PetitPotam is
patched.

**Components:**

- `dfscoerce.py`, `printerbug.py`, `responder.py`
- Module selection logic: try in order, report which succeeded
- Per-module `available()` checks so we never fire a vector that is not
  present

**Exit criteria:**

- At least two vectors work end-to-end in the lab
- Adding a new vector requires only a new file implementing the ABC (no
  orchestrator changes)

**Deliverable:** `autorelay run ... --coercion auto|petitpotam|dfscoerce|...`

## Stage 6 — Opt-in Post-Exploitation

**Goal:** after a relayed session lands, allow the operator to run a
bounded follow-on action.

**Components:**

- `post/ldap_dump.py` — LDAP query as the relayed identity
- `post/smb_exec.py` — command execution if the relayed identity has rights
- `post/secretsdump.py` — explicitly gated; requires
  `--i-know-what-im-doing`
- Post-actions are never automatic; operator confirms each

**Exit criteria:**

- Post-actions run only with explicit flags
- Every post-action logs exactly what it did and against what
- The report includes post-action results with the same
  finding / impact / remediation structure

**Deliverable:** `autorelay run ... --post ldap-dump`

## Stage 7 — Hardening, Docs, Packaging

**Goal:** make it something you would hand to another operator.

**Components:**

- Config file support (`config.yaml`)
- `--dry-run` mode (preflight + plan, no fire)
- README with lab setup instructions
- Operator runbook (what to do when preflight fails, when no session
  lands, when scope is ambiguous)
- Detection notes (what this looks like to Defender / Sysmon, for the blue
  team)

**Exit criteria:**

- A second operator can run it against the lab using only the README
- `--dry-run` produces the full report with zero offensive traffic
  (verified by pcap)

**Deliverable:** v1.0 tag, README, runbook

\newpage

# 4. Milestones

| Milestone | Contents   | Value                                                    |
|-----------|------------|----------------------------------------------------------|
| M1        | Stage 1 + 2 | Standalone preflight / remediation tool — useful on its own |
| M2        | Stage 3 + 4 | End-to-end relay chain with one vector                   |
| M3        | Stage 5 + 6 | Multi-vector, opt-in post-exploitation                   |
| M4        | Stage 7     | Operator-ready v1.0                                      |

M1 is genuinely shippable to a client. If the project stalls after M1, you
still have something valuable.

\newpage

# 5. Risks and Mitigations

| Risk                                             | Mitigation                                                                 |
|--------------------------------------------------|----------------------------------------------------------------------------|
| Impacket API churn breaks the relay wrapper      | Pin Impacket version; isolate all Impacket calls behind one adapter module |
| Tool fires against out-of-scope host             | Hard-code: coercion only to `--target`, relay only to `--relay-targets`; refuse if either is empty |
| Operator runs against production without authorization | Loud banner; `--dry-run` documented; `--yes` required for non-interactive |
| False positive in preflight                      | Prefer "unknown" over "pass" when uncertain; never mark `passed=True` without evidence |
| Post-exploitation surprises                      | Every post-action opt-in, logged, bounded; no auto-chaining                |
| Tool becomes a liability if leaked               | No auto Golden Ticket; no persistence; no cleanup evasion; runs only with explicit scope args |

\newpage

