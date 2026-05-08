# Miinto Companion App — Strategic Re-evaluation
**Date:** 2026-05-05
**Analysts:** Elisa (Hermes Agent) + Claude Code
**Tools used:** Sequential Thinking MCP, document analysis

---

## Executive Summary

The Miinto Companion App project is **strategically sound but tattically ottimista**. The core thesis (self-service removes the team bottleneck) is correct. The 10x growth target is achievable IF the product works and market demand exists. The project has two critical risks that need addressing before Q2 2026 kickoff.

---

## Top 5 Risks Identified

| # | Risk | Severity | Mitigation |
|---|------|----------|------------|
| 1 | **NTS coupling** — building new UI on migrating backend | 🔴 Critical | Decouple at ship time: new partners on POAS v2.5, NTS migrates in background |
| 2 | **DocuSign SLA gap** — partners can delay signing 14 days | 🔴 High | Automated reminder escalation + signing deadline clause |
| 3 | **48h metric unvalidated** — never tested at real scale | 🟡 Medium | Run 20-partner Zapier pilot before Q2 kickoff |
| 4 | **EUR 810M+ GMV claim** — based on unvalidated EUR 150k avg | 🟡 Medium | Finance to validate actual avg partner GMV |
| 5 | **Roles not named** — PO + Engineering Lead missing | 🟡 Medium | Must be named before kickoff, not after |

---

## Metric Change: North Star

**OLD:** "48 hours to onboard"
**NEW:** "**72 hours to first order**"

**Why:** 48h measures onboarding completion. 72h to first order measures value delivery. A partner who completes onboarding in 48h but waits 2 weeks for their first order is a churn risk. The true north star is time-to-value, not process-speed.

---

## NTS Coupling: Decouple Recommendation

Current SUMMIT says "Companion App and NTS are ONE project." **Recommendation: decouple at ship time.**

| Partner Type | Recommended Approach |
|--------------|---------------------|
| New partners (from Q2 2026) | POAS v2.5 + Companion App (ship first) |
| Already on NTS | Full Companion App (early access Q3 2026) |
| Legacy partners | NTS migration continues in background, then Companion App |

**NTS Readiness Gates required before each Companion App phase:**
- Phase 1: NTS dual-run divergence < 0.05% for 14 consecutive days
- Phase 2: NTS dual-run divergence < 0.05% for 30 consecutive days
- Phase 3: All legacy partners migrated before Phase 3 ships
- Schema freeze: 30 days before each release

---

## Pre-Kickoff Checklist (Before Q2 2026)

- [ ] **Pilot:** 20-partner Zapier prototype to validate onboarding flow
- [ ] **Finance:** Validate EUR 150k avg partner GMV (source of EUR 810M+ claim)
- [ ] **Engineering:** Estimate EUR [X] budget and [Y] FTE
- [ ] **Named:** Product Owner (full-time, no split)
- [ ] **Named:** Engineering Lead (full-time, no split)
- [ ] **Signed:** Executive sponsor confirmed
- [ ] **DocuSign:** Add signing deadline clause to new partner contracts
- [ ] **DocuSign:** Implement automated reminder escalation (Hours 2, 4, 8, 24)

---

## KPIs (Updated)

| Metric | Baseline | Target |
|--------|----------|--------|
| Total partners | 600 | 6,000 |
| Onboarding time | 2-4 weeks | 48 hours (to complete) |
| **Time to first order** *(north star)* | TBD | **< 72 hours** |
| Partners onboarded/month | ~20 | 500+ |
| Partner NPS | TBD | 50+ |
| Self-service resolution | 0% | 80%+ |
| Signing completion rate (within 48h) | N/A | >75% |

---

## Files Updated (2026-05-05)

| File | Changes |
|------|---------|
| `Miinto-Partner-Vision-SUMMIT.md` | v1 → v2: Added 72h north star, DocuSign risk, NTS decoupling, readiness gates, pilot recommendation, named roles requirement |
| `03-system-architecture.md` | v2.1 → v2.2: Added NTS Readiness Gates section (§5.3) |

---

## Decision Required from Leadership

1. **Approve NTS decoupling** — new partners ship on POAS v2.5, NTS migrates in background
2. **Fund 20-partner pilot** — near-zero cost, high validation value
3. **Name PO + Engineering Lead** — must happen before Q2 kickoff
4. **Confirm DocuSign signing deadline clause** — requires Legal involvement

---

*Next session: Review pilot results and finalize EUR [X] budget estimate*
