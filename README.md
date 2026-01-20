# Cloudflare Feedback Analyzer

A Cloudflare Workers prototype for aggregating and triaging product feedback using **Workers AI** and **KV storage**, designed to help engineering and product teams react quickly to noisy, fragmented feedback.

---

## Live Demo

**Production Worker**  
https://feedback-analyzer.pif6688.workers.dev

Key routes:
- `/` – Feedback inbox (aggregation)
- `/analyze` – AI-powered triage summary (themes, urgency, value, sentiment)
- `/analyze.json` – Raw analysis output
- `/seed` – Seed sample feedback (POST)
- `/clear` – Clear stored feedback (POST)

---

## Problem

Modern product teams receive feedback from many channels every day:
Customer support tickets, Discord, GitHub issues, email, X/Twitter, community forums, and more.

Because this feedback is **fragmented and noisy**, developers and product managers often struggle to:
- Identify recurring themes
- Understand urgency
- Assess value and impact
- Decide what to act on next

As a result, important feedback is delayed, ignored, or inconsistently handled.

---

## Solution

This prototype is designed not just to analyze feedback, but to support **fast and practical decision-making** for engineering and product teams.

While aggregating feedback and extracting themes is useful, the harder problem is determining **what should be acted on first** when information is incomplete, noisy, and constantly changing.

To address this, the system focuses on two primary decision signals: **urgency** and **value**.

### Urgency

Urgency captures how immediately a piece of feedback requires attention.  
It considers factors such as frequency, severity, and whether the issue blocks core user workflows.

This dimension is intentionally prioritized, as developers and product managers often need to resolve critical issues quickly before investing in longer-term improvements.

### Value

Instead of relying on a simplistic *benefit / cost* ratio, value in this system is defined as a **composite early-stage signal** that approximates potential impact under uncertainty.

Value is derived from four components:

- **Impact** – How much addressing the issue improves user experience
- **Reach** – How many users are likely affected
- **Confidence** – How reliable the signal is, based on feedback consistency and volume
- **Effort (inverse)** – A coarse estimate of implementation complexity

This approach intentionally avoids false precision. Rather than attempting to calculate exact costs or ROI too early, the system provides a **practical prioritization heuristic**, inspired by common PM frameworks (e.g. ICE / RICE), that remains robust even when data is incomplete.

By combining urgency and value, the system surfaces feedback that is not only important, but also **actionable** for engineering teams.

---

## Target Users

This prototype is designed for teams that receive a high volume of fragmented product feedback and need to react quickly.

- **Primary users: Software engineers**  
  Engineers are often the first to feel the impact of noisy feedback, especially when issues are urgent or blocking core functionality.  
  This tool is to help engineers quickly understand *what needs attention now* and *why it matters*, without requiring deep context from multiple feedback sources.

- **Secondary users: Product managers**  
  PMs use the system to coordinate prioritization, validate urgency and value signals, and align engineering efforts with broader product goals.  
  Rather than replacing strategic planning, the tool supports PMs in facilitating faster, more informed execution with engineering teams.

The system intentionally prioritizes **actionability over long-term strategy**, reflecting how feedback-driven decisions are often made in practice.

---

## Key Product Insight 

After submitting or seeding feedback and triggering analysis, users experience the core value of the product:

- **Top Urgency cards** clearly surface what requires immediate attention
- A **Value score** combines impact, reach, confidence, and effort to support prioritization
- Evidence links connect AI insights back to the original raw feedback

The needs moment occurs when fragmented feedback stops feeling overwhelming and instead becomes **clear, prioritized next actions** that engineering teams can immediately act on.

---

The following diagram shows the end-to-end feedback flow and decision boundaries in the prototype:
```text
Feedback Sources
  - Support / Discord / GitHub / Email / Users
                ⬇️
(1) Feedback Input Layer  [Cloudflare Worker]
  - POST /feedback
  - Web form (Inbox UI)
  - Source tagging + dedup (hash id)
                ⬇️
(2) Preprocessing & Normalization
  - Theme inference (heuristics baseline)
  - Priority baseline (P0 / P1 / P2)
  - Guardrails: sentiment/value normalization
  - Robust JSON parsing for model output
                ⬇️
(3) Storage Layer  [Cloudflare KV]
  - FEEDBACK_STORAGE (aggregated inbox)
  - analysis:last cache (optional)
                ⬇️
(4) AI Analysis Engine  [Workers AI]
  - Two-pass analysis
      Pass 1: Themes
      Pass 2: Urgency + Value + Sentiment
  - Retry + deterministic fallback
                ⬇️
(5) Insight & Visualization Layer
  - Inbox triage workflow (status + priority)
  - Triage summary (/analyze) with urgency cards + value score
  - Evidence-linked feedback ids
  - Export: /export.json, /export.csv
```

---

## AI Design & Tradeoffs

Workers AI is used in this system as a **decision-support tool**, not a perfect or authoritative classifier.

In practice, feedback triage and prioritization require rules that are **stable, predictable, and explainable**, especially when the output is used to inform engineering schedules and execution.

### Key Challenges

- **Non-deterministic outputs**  
  Given the probabilistic nature of large language models, identical inputs can produce slightly different results across runs.

- **Occasionally invalid or truncated structured outputs**  
  When generating complex JSON structures, the model may return incomplete or schema-inconsistent results, particularly under token constraints.

- **Variability in value estimation**  
  Value is inherently subjective and context-dependent. Over-optimizing for numerical precision can introduce false confidence and reduce trust.

### Design Tradeoffs

Rather than prioritizing numerical precision, this prototype deliberately emphasizes:

- **Stability** – Similar inputs should produce similar prioritization signals across runs, ensuring predictable behavior in day-to-day usage.
- **Explainability** – Each urgency and value signal is derived from interpretable components (impact, reach, confidence, effort), allowing teams to understand and challenge the output.
- **Actionability** – The system is designed to support scheduling and execution decisions, not to compute exact ROI.

Precision is treated as a **later-stage optimization**. Once outputs are stable, interpretable, and trusted by users, finer-grained accuracy can be iteratively improved without undermining credibility.

This tradeoff aligns with the goal of transforming fragmented feedback into **reliable, actionable prioritization signals**, rather than producing analytically perfect but operationally fragile results.

---

## Local Development

```bash
npm install
npx wrangler dev
