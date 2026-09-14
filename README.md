# HAGGLE

**A domain-expert-curated benchmark based on real-world commodity procurement cases for multi-issue LLM-agent negotiation.**

> **HAGGLE** is a backronym: **H**eterogeneous **A**gent **G**oods **G**ambit, **L**oss-aware **E**valuation — a benchmark for **heterogeneous** multi-issue **A**gent negotiation over real B2B **Goods**, whose strategic **Gambit** is scored with **L**oss-aware feasibility metrics (FD%) that penalize false deals.

[English](README.md) 

---

## Overview

**HAGGLE** is a manually constructed, feasibility-aware negotiation benchmark for agent-to-agent (A2A) LLM bargaining over real Chinese B2B commodity procurement contracts. It accompanies the paper *“Adaptive Opponent Modeling with Acceptance-Aware Offer Selection for Multi-Issue Agent Negotiation”* (AOMAS). We construct a domain-expert-curated benchmark based on real-world commodity procurement cases: each scenario is derived from real brokerage procurement records and annotated by domain experts with private issue priorities, value anchors, and natural-language negotiation tactics for both parties.

The released benchmark contains **68 scenarios across three commodity domains**:

| Domain      | Commodity | Scenarios | Feasible | Infeasible |
| ----------- | --------- | --------: | -------: | ---------: |
| Ferrous     | Rebar     |        20 |       19 |          1 |
| Agriculture | Agri      |        28 |       25 |          3 |
| Energy      | Coke      |        20 |       17 |          3 |
| **Total**   |           |    **68** |   **61** |      **7** |

A scenario is **feasible** when a non-loss-making deal can exist (`buyer.budget ≥ seller.cost`); the 7 infeasible scenarios have no fair deal and a competent agent should **walk away**. This split is what makes the benchmark *feasibility-aware*: utility metrics are computed only over the 61 feasible scenarios (×3 runs = 183 runs), while the **false-deal rate (FD%)** is computed over the 7 infeasible scenarios (×3 runs = 21 runs).

Every scenario is a **4-issue** negotiation with heterogeneous issue types:

| Issue            | Symbol | Type       | Domain                                   |
| ---------------- | :----: | ---------- | ---------------------------------------- |
| Unit price       |  `p`   | continuous | real-valued (CNY/ton)                    |
| Prepayment ratio |  `r`   | enumerated | e.g.  0.20/0.30/0.40/0.50/0.60/0.70/1.00 |
| Payment terms    |  `t`   | enumerated | e.g.  15/30/45/60 days                   |
| Delivery time    |  `d`   | discrete   | e.g.  3/5/7/10 days                      |

Both parties hold **private** additive utilities over these four issues with their own (different) importance weights, so an integrative cross-issue bargain is possible but the opponent’s preferences must be inferred from offers and dialogue.

---

## Scenario schema

Each entry in `*_scenarios.json` is one scenario:

| Field | Meaning |
|---|---|
| `id` | Unique scenario identifier, e.g. `rebar_scenario_001` |
| `commodity` | `rebar` / `agri` / `coke` |
| `spec` | Product specification string (grade, spec) |
| `quantity` | Trade quantity (tons) |
| `seller` | `{cost, prepay_want, terms_max, delivery_min, first_ask}` — seller’s cost, preferred prepay, longest acceptable terms, earliest delivery, opening ask |
| `buyer` | `{budget, target, prepay_want, terms_want, delivery_want}` — buyer’s budget, target price, preferred prepay/terms/delivery |
| `expected_deal` | `{price_range:[lo,hi], delivery_days}` — expert-annotated fair-deal region |
| `context` | Market context narrative (real procurement background) |
| `strategy_buyer`, `strategy_seller` | Natural-language negotiation tactics per party, annotated by domain experts |
| `difficulty` | `Easy` / `Medium` / `Hard` |
| `weights` | Ground-truth issue weights `{p,r,t,d}` for seller and buyer (sum to 1) |
| `weight_rank` | Priority ordering of the four issues for each party |
| `w_S_true`, `w_B_true` | The true private weight vectors used for **evaluation** (god’s-eye utility) |
| `weight_evidence` | Annotation evidence (issue salience counts) used to derive the ranks |

> **Note on private information.** The negotiator agent (seller) only knows its **own** utility `u_N` (i.e. `w_S_true` and its value functions). The partner’s weights `w_B_true`, value anchors, and reservation behavior are **private** and must be inferred from offers and utterances — they are exposed here solely so the evaluation utilities can be recomputed.

---

## Evaluation metrics

### 1. Per-issue value functions (normalized to ~[0, 10], clipped)

**Price** (continuous, self-anchored linear):
```
u_S_price(p) = clip( (p − cost) / (budget − cost) · 10, 0, 10)
u_B_price(p) = clip( (budget − p) / (budget − cost) · 10, 0, 10)
```

**Prepayment** (enumerated, zero-sum mirror):
```
u_S_prepay(ρ) = clip( (ρ − 0.20) / 0.80 · 10, 0, 10)       
u_B_prepay(ρ) = 10 − u_S_prepay(ρ)
```

**Payment terms** (enumerated, zero-sum mirror):
```
u_S_terms(t)  = clip( (60 − t) / 45 · 10, 0, 10)            
u_B_terms(t)  = 10 − u_S_terms(t)
```

**Delivery** (discrete, inverse-urgency, zero-sum mirror):
```
u_B_deliv(d) = clip( (1/d − 1/10) / (1/3 − 1/10) · 10, −10, 10)  
u_S_deliv(d) = 10 − u_B_deliv(d)
```

### 2. Total utility (god’s-eye, ex-post)

Utilities are weighted sums with the scenario’s **true** private weights — not the negotiator’s internal estimate:

```
u_N(o) = u_S(o) = Σ_{j∈{p,r,t,d}} w_S_true[j] · u_S_j(o_j)        # negotiation-agent utility
u_P(o) = u_B(o) = Σ_{j∈{p,r,t,d}} w_B_true[j] · u_B_j(o_j)        # partner-agent utility
TW(o)  = u_N(o) + u_P(o)                                          # total welfare
```

### 3. Feasibility-aware aggregation

Across runs, each scenario is repeated **3 times** and negotiation is capped at **10 rounds**. Let `feas` = (`budget ≥ cost`) and a run “deals” when `accepted=True` with a non-null `final_offer`.

| Metric | Formula | Meaning |
|---|---|---|
| **DR_f** (feasible deal rate) | `\|{feas ∧ deal}\| / \|{feas}\| · 100` | fraction of feasible scenarios that close |
| **u_N^f** (feasibility-aware negotiation-agent utility) | `mean_{feas ∧ deal} u_N` | avg seller utility over feasible deals |
| **u_P^f** | `mean_{feas ∧ deal} u_P` | avg partner utility over feasible deals |
| **TW^f** (feasibility-aware total welfare) | `mean_{feas ∧ deal} (u_N + u_P)` | avg joint utility over feasible deals |
| **AR** (average rounds) | `mean_{feas ∧ deal} rounds_used` | avg rounds to close a feasible deal |
| **FD%** (false-deal rate) | `\|{infeas ∧ deal}\| / \|{infeas}\| · 100` | fraction of infeasible scenarios that still close a loss-making deal (**lower is better**) |

Utility means are taken **only over deals**; walk-aways and timeouts lower DR_f but do not enter the utility average. FD% is the safety signal: it penalizes an agent that accepts a contract below cost or above budget on a scenario where no fair deal exists.


---

## Intended use & limitations

- **Use**: evaluation of multi-issue LLM negotiation agents, opponent-modeling research, feasibility-aware bargaining studies.
- The Chinese strategy/context fields are domain-expert annotations of real procurement tactics; they reflect commercial practice and may contain culturally specific bargaining heuristics.
- Utilities are normalized to a ~[0, 10] scale per issue; absolute magnitudes are comparable within a commodity but calibrated per-domain.

## Citation

If you use this benchmark, please cite:

```bibtex
@dataset{iagentnet_commodity_neg_bench,
  title  = {HAGGLE: A Domain-Expert-Curated Benchmark for Multi-Issue Commodity Negotiation},
  author = {Du, Xinkai and Han, Quanjie and Lv, Chao and Lei, Yao and Sun, Maosong},
  year   = {2026},
  url    = {https://github.com/industryagentnet/iagentnet}
}
```

