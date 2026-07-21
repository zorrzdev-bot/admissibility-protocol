# Security & Sanitisation Notes

This repository publishes the AP-1 evaluation harness so that the results of
run `AP1_FROZEN_20260716_012503` can be independently reproduced, disputed, or
extended (AP-1 v1.2 §8, §10.2). The published `harness.py` is a **sanitised**
version of the internal harness that produced that run.

Two things are deliberately different from the internal harness. **Neither
changes any score, prompt, model setting, or classification rule.** They are
documented here in full so the sanitisation itself is auditable.

## 1. System-under-test authentication is withheld

The internal harness called the system under test ("PILVI") through a
production API behind an authenticated evaluation endpoint. The concrete
mechanism — the endpoint URL, the per-profile bearer tokens, and an internal
evaluation header — is **operational access to a live system**, not part of
the measurement, and is not published.

In `harness.py`, the system-under-test adapter (`run_system_under_test`):

- reads its endpoint from `SUT_ENDPOINT` (a placeholder that must be configured
  explicitly; the harness will not silently contact any host);
- uses only a **generic bearer token** read from the environment, if present;
- contains **no** internal evaluation/bypass header.

The **call shape** (request body, response normalisation, multi-turn history
handling) is preserved so the logic is fully auditable. Any party reproducing
AP-1 supplies their **own** system-under-test adapter by implementing that one
function; the rest of the harness — context construction, the five-way
classifier, logging — is system-agnostic.

> Why this is not a fairness problem: the withheld details govern *how the
> harness reaches the system under test*, not *what the system was asked* or
> *how its answers were scored*. Every prompt, setting, and scoring rule is
> published verbatim (AP-1 §5.4), and every raw response is in the run log.

## 2. The calculator tool uses a safe AST evaluator (no `eval()`)

The calculator tool offered to the three comparator models originally evaluated
expressions with an allow-listed `eval()`. The published harness removes
`eval()` entirely and instead parses each expression to an abstract syntax tree
and walks **only** an explicit set of node types (numeric constants, the
arithmetic operators `+ - * / // % **`, unary `+/-`, the constants `pi`/`e`, and
the functions `abs round min max sum pow int float sqrt ceil floor log log10`).

The set of permitted operations is the **same** as the internal allow-list, so
tool behaviour is identical for every arithmetic expression the comparators
produced. A parity check against the original allow-listed evaluator on the
expression shapes that occurred in the run returns identical results, and an
offline self-test (`python harness.py --selftest`) verifies both correctness
and that malicious inputs — attribute traversal, imports, `open`, nested
`eval` — are refused.

> Why this matters for AP-1 specifically: publishing `eval()`-based code in the
> reference implementation of a standard about *not letting probabilistic
> systems hand arbitrary expressions to something that executes them* would be
> self-undermining. The safe evaluator removes that surface entirely.

## What is **not** sanitised (published exactly as run)

Everything that bears on the result is published unchanged:

- both shared system prompts, verbatim;
- the comparator model IDs and their settings, **including the rationale** for
  each (Sol `reasoning_effort="medium"` on every call — the "B12" note; Fable
  default adaptive thinking; Gemini defaults — a preview model, see AP-1
  §Limitations);
- the per-provider tool loops (OpenAI Responses, Anthropic Messages, Google
  GenAI), including provider-specific quirks;
- the five-way classifier (`CLASSIFIER-REFUSED` / `MODEL-DECLINED` / `COMPUTED`
  / `RETRIEVED` / `FABRICATED`) with tool invocation logged as a **separate
  axis** (the D7 finding);
- the deliberate **non-sanitisation of the data-embedded injection payload**:
  account names, including the profile-P5 injection string, are passed to the
  comparators verbatim — testing that behaviour is the point of the test.

## Data

The evaluation profiles (`_eval_profiles_v6.json`) and per-account contexts are
**synthetic** (Plaid Sandbox); they contain no real customer data. Any
access token field present in a profile is a synthetic/sandbox value, not a
live production credential.

## Reporting a vulnerability

If you believe you have found a way for the calculator evaluator to execute
anything beyond the permitted arithmetic, or any other security issue in this
harness, please open a private security advisory on this repository (or contact
the maintainer) rather than a public issue. We will confirm, fix, and credit.

---

*This document accompanies the sanitised `harness.py`. The unsanitised internal
harness is not published because it embeds live-system access; the sanitisation
above is the only difference, and it is auditable line-for-line against this
description.*
