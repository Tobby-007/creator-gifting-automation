# Creator Gifting Automation: Production Claude API Integration

**Status:** Live in production since February 2026
**Role:** Sole engineer
**Employer:** Triquetra Health

> This is a case study describing proprietary work. Production code is not published. For code demonstrating the underlying AI-plus-fallback pattern, see the [claude-scoring-fallback](../claude-scoring-fallback) sample repository.

## The Problem

Triquetra Health runs a creator gifting program. Content creators apply through an intake form, and if approved, they receive a curated bundle of products (worth roughly $60 to $200 depending on fit) as a $0 gift order in exchange for organic content.

The manual workflow had three steps, all human-owned:

1. **Application review:** a marketing associate reads each application, checks the creator's profile against brand-fit criteria (audience size, content style, geographic market, existing partnerships), and decides approve or reject
2. **Product selection:** for approved creators, choose 3 to 5 products from a 411-item catalog that fit the creator's niche (skincare vs supplements vs beauty), audience demographics, and stated preferences
3. **Order orchestration:** create a $0 Shopify order using the internal Staff Order Request app, ship it, and send an approval email with unboxing guidelines

At steady state this took 15 to 25 minutes per application. When a marketing campaign spiked applications (150+ per week during launches), the queue would fall behind by 2 to 3 weeks. Some approved creators lost momentum and never posted.

Additionally, 89 applications had been queued and neglected during a company-wide pivot period. Someone had to work through the backlog.

## The Approach

I built an AI-orchestrated pipeline that scores applications, selects products, and orchestrates $0 Shopify orders, with human-in-the-loop review before dispatch.

### Pipeline Stages

**Stage 1: Application ingestion.** New submissions from the intake form (a Typeform integration writing to a Google Sheet) trigger the pipeline. Each application includes the creator's platform handles, follower counts, content niche, geographic market, and free-text answers to brand-fit questions.

**Stage 2: Claude API scoring.** The application is sent to the Anthropic Claude API (Claude Haiku model) with a structured prompt that asks the model to score the applicant on five criteria: brand fit, audience relevance, content quality, geographic market alignment, and reliability signal. Response is required in strict JSON. The scoring prompt includes 12 few-shot examples curated from previously-approved and previously-rejected applications.

**Stage 3: Product selection.** For applications scoring above the approval threshold, a second Claude call selects 3 to 5 products from the 411-item catalog. The catalog is passed to Claude as a structured list with product name, category, target demographic, and current inventory status. The model returns an ordered list of product IDs with reasoning for each pick.

**Stage 4: Order orchestration.** Selected products are packaged into a $0 Shopify draft order using the Shopify GraphQL Admin API. The order is created with a specific tag (`creator_gift_pending_review`) so it does not auto-fulfill.

**Stage 5: Human-in-the-loop review.** A Slack message posts to the marketing channel with the creator profile, Claude's scoring rationale, the selected products, and two buttons: Approve (converts draft to fulfilled order) and Reject (deletes draft and sends decline email). No order ships without human approval.

**Stage 6: Structured audit logging.** Every stage writes to a Sheets tab: application ID, Claude scores, selected products, human decision, actor, latency at each stage. This is the debug artifact when anything looks wrong.

### Fallback Behavior

**If the Anthropic API is unavailable** (timeout, 5xx, or unexpected response format), the pipeline falls back to a rule-based scoring layer using explicit follower count thresholds, geographic market rules, and category matching. The rule-based fallback is deliberately conservative: it will approve fewer applicants than Claude would, and it flags every fallback decision for human review.

The fallback exists because Claude API downtime should not stop the pipeline from processing applications. Better to process fewer with rules than none at all.

## Design Decisions Worth Naming

**Claude Haiku, not Sonnet or Opus.** The task is high-volume, structurally repetitive, and does not require deep reasoning. Haiku's speed and cost profile fit the task shape. Testing Sonnet showed marginal quality improvement at 5x the cost per application.

**Strict JSON response required.** The prompt explicitly requires JSON output with a schema. The parser validates the response structure before accepting it. Malformed responses trigger a single retry with a clarifying prompt, then fall back to rule-based scoring.

**Human review is non-optional.** The pipeline never ships an order without human approval. This is a deliberate constraint. AI-driven order creation without human review would compound errors and create brand risk.

**Rule-based fallback is not aspirational.** It is used in practice when the Anthropic API is slow or unavailable. It has processed real applications. It exists as production code, not as a placeholder.

**Audit logging captures both AI and human decisions.** When a human overrides Claude's recommendation (approves an application Claude scored below threshold, or rejects one Claude scored above), that override is logged with a required reason field. Over time this creates training data for prompt refinement.

## Tech Stack

- **Runtime:** Google Apps Script (V8 engine)
- **AI:** Anthropic Claude API (Haiku model, JSON-mode-equivalent via strict prompting and schema validation)
- **APIs:** Shopify GraphQL Admin API, Slack Web API, Google Sheets API
- **Storage:** Google Sheets (audit log, catalog, prompt few-shots)
- **Auth:** Anthropic API key (Script Properties), Shopify OAuth 2.0, Slack Bot Token
- **Trigger:** Scheduled every 15 minutes; also on Typeform webhook

## Impact

- **89 backfilled applications processed** in the first week after deployment (previously 4 to 6 weeks of manual work)
- **Ongoing volume** of ~30 to 60 applications per week processed without queue backlog
- **~12 to 20 hours per week** of marketing associate time reclaimed
- **Time-to-decision reduced from days to hours** for typical applications
- **Zero mis-shipped gift orders** since deployment (human-in-the-loop caught two Claude misfires that would have shipped inappropriate product bundles)

## Lessons Learned

**1. Strict JSON validation is worth the extra code.**
Claude generally returns clean JSON when prompted correctly, but "generally" is not "always." A defensive parser with schema validation catches malformed responses before they crash the pipeline. Two malformed responses in the first month, both caught cleanly.

**2. Human-in-the-loop is not a limitation. It is the design.**
Fully autonomous order creation would have shipped faster, but any single Claude misfire would have shipped a product bundle worth $60 to $200 to the wrong creator. The human review step adds ~30 seconds per decision and catches every misfire.

**3. Fallback code has to work.**
The rule-based fallback started as aspirational. During an Anthropic API incident in April, it processed 22 applications with only one flagged as ambiguous. That day validated the design choice. If the fallback had been a placeholder, the pipeline would have failed.

**4. Few-shot examples matter more than prompt cleverness.**
The first version of the scoring prompt was long, elaborate, and full of criteria explanations. It performed worse than a much shorter prompt with 12 hand-curated few-shot examples pulled from real approved and rejected applications. Real examples out-explain rules every time.

**5. Structured audit logging enables prompt iteration.**
When a human overrides Claude, the reason gets logged. Every month I review overrides and update the few-shot examples or scoring prompt. This is a slow feedback loop that keeps improving output quality without any model change.

## What's Next

- Add tier-based product selection (Silver, Gold, Platinum tiers based on scoring band)
- Extend to influencer partnerships (paid gifting with contract flow) using the same core pipeline with an added contract-generation stage
- Build a monthly performance dashboard that correlates Claude scores against actual content output (post volume, engagement, conversion)

## References

- Anthropic Claude API: [https://docs.claude.com](https://docs.claude.com)
- Shopify Draft Orders: [https://shopify.dev/docs/api/admin-graphql/latest/mutations/draftOrderCreate](https://shopify.dev/docs/api/admin-graphql/latest/mutations/draftOrderCreate)
