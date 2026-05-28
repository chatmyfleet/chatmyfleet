# Business Research: ChatMyFleet

> Loop 1 business research. Distils the locked v6 narrative in `brief/chatmyfleet/` into the Orbit `research-business-tmpl` shape so downstream loops have a single artifact to cite. Public-voice only — no internal strategy vocabulary (Track A/B, Mode 1, kill criterion, week-N milestones, revenue projections).

Last updated: 2026-05-28
Source documents: `brief/chatmyfleet/product.md`, `brief/chatmyfleet/ux.md`, `brief/chatmyfleet/oss-manifesto.md`, `brief/chatmyfleet/affiliates.md`, `brief/chatmyfleet/maintenance.md`

---

## Business Goal & Vision

ChatMyFleet is the open-source chat layer that lets AI builders hand day-to-day agent operation to their clients' ops teams. The builder ships an agent (Lindy, OpenAI Assistants, LangGraph, n8n, Dify, Flowise, custom Python — anything), wires it through one of ChatMyFleet's connectors, deploys to the client's Slack workspace, and exits the operational loop. The client's marketers, salespeople, and fulfillment staff approve agent actions inline, talk to agents via `@mention`, and teach the agent new rules in chat. The builder stays available, not entangled.

The core problem ChatMyFleet exists to solve: **the builder is the operational bottleneck.** Every refund question, every policy clarification, every "why did it do that?" routes back through the person who built the agent. That bottleneck caps a solo AI builder or 1-5 person agency at 3-5 client deployments. ChatMyFleet removes the bottleneck by giving the client's team a chat-native operator surface they can actually use — with HITL approval, append-only audit, and a memory layer that survives an agent-platform swap.

The market does not currently ship an operator-facing chat layer that wraps any agent platform under a license that's safe to deploy commercially per client. Slack Agentforce locks you to Salesforce. Microsoft Copilot Studio locks you to M365. HumanLayer ships an SDK with no operator UI. Lindy and Dify ship the agent IDE but assume the builder is the user. ChatMyFleet sits in that gap — the **operator inbox for non-builders**, BYO-everything, Apache 2.0.

---

## Target Audience (Persona)

Three audiences. The product is the same for all three; the surface differs.

### Builder (the primary integrator)

- **Who:** Solo AI builder or 1-5 person AI agency that has already shipped at least one production agent.
- **Where they live:** GitHub, dev.to, LinkedIn, founder Twitter, Slack communities for the agent platform they use.
- **Stack signal:** comfortable with `docker compose up`, Slack app config, agent-platform dashboards. Already pays for an OpenAI/Anthropic key.
- **Pain points:**
  - Stuck as on-call for every client's operational question
  - Can't pitch to a 4th or 5th client without burning out
  - Has built the agent but doesn't have a way to hand operation over
  - Doesn't want to build a custom chat UI per client
- **Needs:**
  - 5-minute self-host install (Docker Compose, one tenant per client)
  - One integration that covers their existing agent platform
  - Apache 2.0 so it's safe to deploy commercially in client engagements
  - HITL + audit so the client can self-serve sensitive actions without the builder in the loop
- **Operating reality:** likely runs 2-10 ChatMyFleet deployments in parallel — one per client.

### User Admin (the client's ops lead)

- **Who:** COO, CTO, or head of ops at the client SMB or mid-market org.
- **Where they live:** Slack day-to-day; Web UI for the 5% of admin work that needs visual or bulk operations.
- **Pain points:**
  - The agent the consultant built works, but the team can't operate it without pinging the consultant
  - No visibility into what the agent actually did or why
  - No way to set policies or approval thresholds without an engineering ticket
- **Needs:**
  - Bulk/visual admin (member management, audit search, policy editing) in a Web UI
  - Confidence that the audit log is real (append-only, exportable, links back to chat threads)
  - Ability to teach the agent rules in natural language ("auto-approve refunds under $200 for repeat customers")
  - Email fallback when the on-call approver is offline

### Staff operator (front-line ops)

- **Who:** Sales rep, support agent, fulfillment ops, SMB owner running their own shop, agency account manager.
- **Where they live:** Slack only. Never opens the Web UI.
- **Pain points:**
  - Agent fires actions they don't understand
  - Approval requests with no context ("Approve this? [Y/N]")
  - Can't tell which agent did what when something goes wrong
- **Needs:**
  - Approval prompts with full context (what the action is, which policy triggered HITL, what happens on Y vs N)
  - `@mention` access to ask agents simple questions
  - Inline `[Approve] [Decline]` buttons, no context switch
  - About 95% of all operations land on this surface.

### Version-zero adopter

The founder, deploying ChatMyFleet for an existing AI sales agent already running in production for a Thai SMB client. Real refund and discount approvals run through HITL on launch day. The product is designed against a live production constraint, then generalised into OSS — not the other way around.

---

## Market Opportunities

**Why now (2026):**

1. **Agent platforms have matured.** OpenAI Assistants is GA. LangGraph, Mastra, Lindy, Dify, Flowise, n8n are all production-deployed at scale. The agent-building layer is no longer the bottleneck — operating the agent is.
2. **MCP is becoming the standard.** The Model Context Protocol gives ChatMyFleet a single zero-line integration that covers 200+ MCP servers and most of the LangGraph/Mastra/OpenAI Agents SDK ecosystem. v0.6.0 unlocks this.
3. **AI consultancies are scaling into the bottleneck.** Anecdotally, the "I built one agent, now I'm permanent on-call" failure mode is widely reported in AI-builder communities. The market is currently solving it ad-hoc (custom Slack apps per client), which doesn't scale.
4. **OSS distribution still compounds.** Apache 2.0 + Docker + GitHub remains the most efficient distribution stack a solo developer has. No sales team needed.

**Market shape:** demand-pull from a small but growing population of AI builders shipping to clients. Not a mass market. Adoption is paced by builder demand signal (issues, sponsors), not by a marketing engine.

---

## Competitor Analysis

The market gap ChatMyFleet occupies: **operator inbox for non-builders, BYO everything, Apache 2.0.** No single competitor covers all three.

| Competitor | Key Features | Strength | Weakness | Pricing Model |
|---|---|---|---|---|
| **Lindy / Dify / n8n / Flowise / LangGraph** | Build + reason + UI in one stack; chat is a test/end-user surface | Polished agent IDE; large existing user bases | Assume the builder is the daily user; no operator-grade audit linkback; no cross-platform memory | Mix of OSS + paid tiers; some affiliate-eligible |
| **Slack Agentforce / Salesforce Einstein Agent** | Chat-first agents inside Salesforce | Native to Salesforce; enterprise security | Locked to Salesforce stack; not BYO-agent; commercial license | Paid (Salesforce per-seat) |
| **Microsoft Copilot Studio + Teams** | Chat-first inside M365 | Native to Teams; large enterprise distribution | Locked to M365; not BYO-LLM; commercial license; no chat-linked audit | Paid (M365 add-on) |
| **Front / Intercom** | Polished operator inbox UX | Mature inbox design, multi-channel | No agent concept; built for human-operator workflows; no HITL primitive | Paid SaaS per-seat |
| **HumanLayer** | Approval-as-a-service SDK | Solid HITL primitive | SDK only — no operator UI, no chat surface, no memory, no audit | Paid SaaS |
| **Letta / Zep** | Agent memory layer | Strong semantic memory | Memory is for agents, not operator-readable/editable; no chat or approval surface | Paid SaaS / OSS mix |
| **Langfuse / LangSmith / Helicone** | Agent observability + traces | Developer-grade tracing | Built for developers; not an operator audit log; no chat linkback | Paid SaaS / OSS mix |

### Adjacent OSS shape comparables (governance + funding model, not feature competitors)

- **Datasette** (Simon Willison) — solo BDFL, Apache, Sponsors-funded
- **Livewire / AlpineJS** (Caleb Porzio) — solo BDFL, MIT, Sponsors-funded
- **SQLite** (Hipp + small team) — solo-ish, Public Domain, long-lived
- **Sindre Sorhus's packages** — solo, many libraries, Sponsors + Tidelift

These projects validate that the "single maintainer, permissive license, sponsorship-funded, long horizon" shape is sustainable for 5+ years when the product is focused.

### Where ChatMyFleet's wedge sits

| Axis | ChatMyFleet | Closest competitor |
|---|---|---|
| Operator UI on top of arbitrary agents | Yes | None (HumanLayer is SDK-only; Front/Intercom have no agent concept) |
| BYO agent platform | Yes (1 connector at v0.4, 8 by v0.11) | Agentforce/Copilot Studio are stack-locked |
| BYO LLM key | Yes | Most competitors meter LLM usage |
| Audit log linked back to chat thread | Yes — the differentiator | None of Lindy, Dify, n8n, Flowise, Copilot Studio do this |
| Memory survives agent-platform swap | Yes (memory in CMF, not the agent) | Letta/Zep store memory but not operator-readable |
| Apache 2.0, no open-core carve-out | Yes | n8n/Dify have OSS + paid tiers; most others are commercial-only |
| Solo BDFL with 5-year maintenance commitment | Yes — public signed commitment | Datasette/Livewire model; not common in agent tooling |

---

## Potential Revenue Model

ChatMyFleet is OSS-funded patronage, not a SaaS. Two transparent revenue paths, neither paywalls features.

### Path 1 — GitHub Sponsors (patronage)

Four tiers, monthly subscription, no feature gating:

| Tier | Price | Tangible benefits |
|---|---|---|
| Quiet support | $5/mo | THANKS.md listing, monthly transparency post access |
| Builder support | $25/mo | + 24h early access to dev.to posts, best-effort issue priority |
| Production user | $100/mo | + optional 30-min monthly call, DM line for production questions |
| Sustained backer | $500/mo | + README backer listing, quarterly 60-min strategic conversation |

Sponsors do NOT unlock: paywalled features, SLA upgrades, "Pro" version, roadmap control. Patronage = keeping OSS alive; the product is identical for sponsor and non-sponsor.

### Path 2 — Agent-platform affiliates

Small commissions when builders sign up for the agent platforms ChatMyFleet integrates with via integration-page links. **Locked Wave 1 cohort (in build order): n8n → Dify → Lindy → Flowise.**

| Partner | Commission | Window | Integration ships |
|---|---|---|---|
| n8n | 30% | 12 months | v0.7.0 (~Nov 2026) |
| Dify | 30-50% | 12 months | v0.8.0 (~Dec 2026) |
| Lindy | 30% of subscription | 24 months | v0.9.0 (~Jan 2027) |
| Flowise | 20% | 12 months | v0.10.0 (~Feb 2027) |

Subsequent cohorts (Wave 2: Make.com, Botpress, Relevance AI; Wave 3: Voiceflow, MindStudio, CustomGPT.ai, AgentCenter.cloud) sit as unlocked planning queues; order shifts on builder demand. Integrations that earn zero (OpenAI Assistants, MCP, generic webhook, custom Python) are equally first-class — selection is technical, not commercial.

### Deferred / explicitly NOT in the revenue model at launch

- **No managed hosting** at v0.4.0. Year 2+ consideration only if demand funds the operational work; if it ever ships, it runs the same OSS code.
- **No "Pro" tier.** Apache 2.0 across the whole repo, no open-core carve-out, no source-available rug-pull.
- **No paid feature gates** ever — features that ship under Apache 2.0 stay under Apache 2.0.
- **No enterprise add-ons** (SSO, advanced RBAC, audit retention >30 days) until a paying customer signs a contract; if added, they'd ship in a separate `chatmyfleet-enterprise` repo on a different license, and the Apache 2.0 OSS stays unchanged.
- **No relicensing** under any acquisition or funding pressure. Locked.

### Transparency mechanics

Monthly transparency posts from M4 (Oct 2026) onward publish:
- GitHub Sponsors total + new signups + churn
- Affiliate revenue (gross) — itemised per partner
- Decisions made this month (shipped, declined, still considering)
- One thing learned

Numbers are real and published. Zero months are reported as zero.

---

## Strategic Recommendation

**Go.** Proceed to Loop 2/3 against the locked v0.4.0 scope.

### Why this clears the "Winning Zone" bar

1. **Real production constraint Day 1.** The founder's existing Thai SMB client runs live refund/discount approvals through HITL on launch day. The product is being designed against a real workload, not a hypothetical one.
2. **Defensible market gap.** The "operator inbox for non-builders, BYO-everything, Apache 2.0" position is not currently occupied. Each adjacent competitor misses at least two of the three axes.
3. **Sustainable funding model.** Patronage + affiliates pattern is proven by Datasette, Livewire, AlpineJS. Doesn't require sales, marketing, or VC.
4. **Distribution stack matches solo capacity.** Apache 2.0 + GitHub + Docker requires no sales team, no marketing budget, no trial-credit-card infrastructure. The repo and the 5-minute install are the entire go-to-market path.
5. **Architectural calls de-risk launch.** Three non-negotiable decisions (gateway/worker split, Postgres-as-SoT, append-only audit via Postgres role permission) remove the most common categories of late-stage rework.

### Where the strategy is most exposed (risks worth naming)

1. **Solo bandwidth.** ~14.5 engineer-weeks of work in a 12-week window (1.5× AI-assisted throughput). ~2 weeks of buffer. Slippage past Week 12 puts the dated launch at risk. Mitigation: epic sizing discipline in Loop 1, ruthless defer-list enforcement, micro-task test gate to prevent rework.
2. **MVP-tier surfaces ship without sad-path coverage.** Web UI polish, memory layer breadth, and slash command UX iterate post-launch. If an early adopter hits a sad-path early, the trust impact is disproportionate. Mitigation: monthly minor cadence + responsive issue triage + transparent CHANGELOG.
3. **Affiliate revenue is back-loaded.** First affiliate-eligible integration (n8n) ships at v0.7.0 (~Nov 2026). Until then, Sponsors is the only revenue path. Mitigation: launch with Sponsors page live + Sponsors tier outreach to early adopters; affiliate revenue accrues as integrations land.
4. **Spike risk on 9 named technical doubts.** All 9 carry into `spec.md` as Pending spikes (6 from the brief's open doubts + 3 added during the Loop 1 architecture review to de-risk the foundation epics: audit-role enforcement, distroless packaging, Redis Streams restart-safety). Resolved per-epic during Loop 4 pre-flight when credentials are available (Slack workspace, OpenAI key, SMTP provider); Spikes 7–9 need no external credentials (local Postgres/Redis/Docker only). Mitigation: the foundation epics (1, 2, 3) each open with a de-risking spike; no Epic can be marked `Approved` until its dependent spikes are resolved.

### Conditions for "Go"

- User approves `spec.md` (Loop 1 exit).
- 9 spikes scheduled into Loop 4 pre-flight per dependent epic.
- v0.1.0 trust-signal artifacts (LICENSE, TRADEMARK, SECURITY, CONTRIBUTING, MAINTENANCE, CHANGELOG, AFFILIATES, Sponsors page, `chatmyfleet.com` minimal landing) populated by Week 4 when the repo flips public.
- Monthly minor release cadence (first Friday) and monthly transparency posts (M4 onward) committed in `MAINTENANCE.md` and the project shape supports holding it.

Proceed.
