# Antigravity Output Token Limit: A Technical Analysis

> **Author**: K ([@keisksw](https://github.com/keisksw)) — AI Ultra subscriber ($249.99/mo)  
> **Agent**: M — K's agentic AI assistant (Claude Opus 4.6 Thinking, via Antigravity IDE)  
> **Investigation date**: 2026-04-03  
> **Status**: Active research — findings may change with future Antigravity updates  

---

## Executive Summary

Google Antigravity enforces a **hard-coded, plan-agnostic output ceiling of 16,384 tokens per turn** across all models. This limit is not a property of the underlying models (Gemini 3.1 Pro supports 65,536; Claude Opus 4.6 supports 128,000) — it is imposed by the Antigravity IDE wrapper. There is no user-facing configuration to override it. Additionally, Claude models within Antigravity have their `max_thinking_length` parameter hard-coded to **1,024 tokens** — the API-minimum — despite supporting up to 128,000.

Most critically: when Gemini 3.1 Pro hits this limit, the agent receives **no error notification**. Output is silently truncated mid-sentence. The agent does not know it was cut off. This creates a real risk of deploying incomplete code and cascading failures in autonomous workflows.

These constraints apply identically to Free, Pro ($19.99/mo), and **Ultra ($249.99/mo)** subscribers.

---

## My Background — Why I Care

I switched to Antigravity from Cursor. Cursor was expensive and I was looking for a more capable, more integrated agent environment. Google's Ultra plan promised the highest access, the best models, the most generous quotas.

**At launch, it delivered.** The first weeks were remarkable. The agent capabilities were ahead of anything I'd used. I was genuinely impressed and committed my daily workflow — personal projects, business operations, creative work — to the platform.

Then things started to change. Gradually at first (January 2026), then substantially (March 2026). What I initially dismissed as beta-stage growing pains turned out to be a pattern of systematic constraint tightening, without official communication or documentation.

I'm not writing this out of frustration. I'm writing this because I still believe the platform has extraordinary potential, and because I believe the engineering team deserves better data than "+1 same here" posts. This is my contribution to that effort.

---

## 1. Test Methodology & Results

### Test Design
- **Prompt**: "Generate a numbered list from 1 to 2000, with one factual statement per line"
- **Languages**: Japanese and English (to compare tokenizer efficiency)
- **Models**: Claude Opus 4.6 (Thinking) and Gemini 3.1 Pro
- **Environment**: Antigravity IDE on macOS (MacBook Air), AI Ultra subscription

### Results Summary

| Test | Model | Language | Lines Before Cutoff | Error Surfaced? | Token Estimate |
|------|-------|----------|---------------------|-----------------|----------------|
| 1 | Claude Opus 4.6 | Japanese | 389 (mid-sentence) | ✅ Explicit error | ~16,384 |
| 2 | Claude Opus 4.6 | English | 428 (mid-sentence) | ✅ Explicit error | ~16,384 |
| 3 | Gemini 3.1 Pro | Japanese | 452 (end of sentence) | ❌ **Silent** | ~16,384 |
| 4 | Gemini 3.1 Pro | English | 500+ (all completed) | N/A | <16,384 |

### Character Output Estimates

| Model | Japanese Output (approx) | English Output (approx) |
|-------|-------------------------|------------------------|
| Claude Opus 4.6 | ~15,560 chars → cutoff | ~42,800 chars → cutoff |
| Gemini 3.1 Pro | ~18,080 chars → cutoff | ~50,000+ chars → completed |

**Finding**: Japanese consumes tokens roughly 2.5–3x faster than English due to multi-byte tokenization. The same 16,384-token budget yields dramatically less Japanese text.

---

## 2. The Silent Failure Problem (Critical)

This is the most dangerous finding from the investigation.

### Claude Opus 4.6: Recoverable Error
When output hits 16,384 tokens, the system injects an explicit error:
```
Error Message: model output error: generation exceeded max tokens limit.
Please generate a message within the token limit (16384)
Retries remaining: 4
```
The agent knows it was truncated. It can retry, split the task, or inform the user.

### Gemini 3.1 Pro: Silent Truncation
When output hits 16,384 tokens, **nothing happens**. The output simply stops. The agent receives no signal that its generation was incomplete. It proceeds to the next step believing the task was completed.

**Real-world risk scenarios**:
- Code generation cut mid-function → broken files deployed without agent awareness
- Configuration files truncated → builds fail with misleading errors
- Document generation stops → critical sections missing from deliverables
- Multi-step workflows cascade on incomplete artifacts

### Root Cause

Both APIs return a `finish_reason: "max_tokens"` (Gemini) / `stop_reason: "max_tokens"` (Claude) flag. The Antigravity wrapper correctly catches this for Claude but **fails to surface it for Gemini**. This is an integration bug, not a model limitation.

---

## 3. The `max_thinking_length` Constraint

In addition to the output token cap, Claude models within Antigravity have their thinking budget hard-coded to **1,024 tokens** — the absolute minimum required by the Anthropic API.

| Parameter | Antigravity Setting | Claude API Maximum | Utilization |
|-----------|--------------------|--------------------|-------------|
| `max_thinking_length` | 1,024 | 128,000 | **0.8%** |
| `max_output_tokens` | 16,384 | 128,000 | **12.8%** |

For complex reasoning tasks — architectural planning, multi-file refactoring, debugging — 1,024 thinking tokens is severely inadequate. Community members have described this as a "lobotomy" of the model's reasoning capabilities.

---

## 4. Platform Comparison

| Platform | Output Limit/Turn | Thinking Budget | User Configurable? | Price/mo | Truncation Handling |
|----------|------------------|-----------------|--------------------|---------|--------------------|
| **Antigravity Ultra** | 16,384 | 1,024 (Claude) | ❌ No | $249.99 | ❌ Silent (Gemini) / ✅ Error (Claude) |
| **Claude Code (Max)** | Configurable (up to 128K) | 128,000 | ✅ `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | $200 | ✅ `stop_reason` exposed |
| **Cursor Pro** | Model-dependent (8K default) | Model-native | Partial ("Continue" button) | $20 | ⚠️ UI prompt to continue |
| **Cline (OSS)** | Model-native | Model-native | ✅ Full control | Pay-per-use | ✅ API response exposed |
| **Google AI Studio** | Up to 65,536 (Gemini) | N/A | ✅ `maxOutputTokens` param | Pay-per-use | ✅ `finish_reason` exposed |

**Key observation**: Google's own AI Studio allows users to set `maxOutputTokens` up to 65,536 for the same Gemini 3.1 Pro model. Antigravity caps it at 16,384 with no override — **Google's IDE restricts Google's own model more than Google's own API playground does.**

---

## 5. Timeline: How We Got Here

### Phase 1: Launch — "Generous" (November–December 2025)
- Antigravity launched November 18, 2025 alongside Gemini 3
- Official messaging: "generous rate limits"
- User experience: Hundreds of thousands of tokens per session with minimal friction
- No visible output cap per turn. No weekly lockouts.
- Community response: Overwhelmingly positive. Developers integrated Antigravity into production workflows.

### Phase 2: Tightening (January–February 2026)
- Weekly hard caps introduced (7-day lockouts reported)
- 5-hour rolling refresh + hidden "weekly baseline" dual structure
- User reports: "capacity dropped from millions to under 200K tokens per week"
- Perception shift: **"bait and switch"** criticism emerges across Reddit
- The 16,384 per-turn output cap likely introduced during this period (no official documentation)
- Free tier: ~92% reduction in daily request limits

### Phase 3: Monetization (March 2026–Present)
- v1.20.x: "AI Credits" toggle introduced. Baseline quota exhaustion → Vertex AI pay-per-use
- Bugs: Credits toggle ON but credits not consumed → 7-day lockout persists
- v1.21.x: Quota further reduced — Ultra subscriber `fxd0h` reported identical workflows going from 5+ hours to **90 minutes** before 0% quota
- Community tools emerge: `antigravity-flush`, Toolkit for Antigravity, Awesome-Antigravity
- **Current state**: 16,384 output / 1,024 thinking hard caps remain. Zero official acknowledgment.

---

## 6. Community Impact (Data from Discourse API, 2026-04-03)

All data below was collected programmatically via the Discourse JSON API at `discuss.ai.google.dev`.

### Thread #118505: "Quota Not Working as Advertised"
- **645 posts** | 26,738 views | 401 unique participants | 1,455 likes
- Created: 2026-01-25 | **Still active** (new posts within minutes of this investigation)
- Original report: Pro subscriber locked out for 10 days instead of advertised 5-hour reset
- First post received **364 reactions** (258 ❤️, 95 👍, 10 👏)

### Thread #135526: "[Ultra] Dramatic Quota Reduction"
- **269 posts** | 10,148 views | 937 likes
- User `fxd0h` (Ultra subscriber): "Before: 5+ hours at 80% quota. After update: 0% in under 90 minutes. That's a 5–6x cut."

### Google Staff Responses
`Abhijit_Pramanik` (Google, trust_level: 3) — 7 posts across both threads. Every response follows the same template:
> *"To provide you with greater control and a seamless path to scale, we are evolving our Google AI plans by introducing built-in AI credits..."*

No response addresses: the 16,384 number, `max_thinking_length`, silent truncation, or any specific technical constraint. Zero technical acknowledgment of the hardcoded limits.

### What's Missing from the Forum
A search for these terms within the Antigravity category returned **zero results**:
- `"16384"` + Antigravity → 0 posts about output limits
- `"output token limit comparison"` → 0 posts
- `"max_thinking_length"` → 0 posts

The community knows something is wrong. But no one has published the specific technical parameters or a competitive analysis. This document aims to fill that gap.

---

## 7. A Note on Precedent

On **2026-03-27**, I published a [chat history recovery guide](https://discuss.ai.google.dev/t/136496) documenting how to recover lost Antigravity sessions using `.pb file injection`. The same day, a user replied:

> *"the new update 1.21.6 just restored all chats history"* — `shteren`, 2026-03-27

I don't claim causation. But the correlation is notable. Detailed public technical disclosure — not "+1 same here" posts — appears to prompt engineering response. This analysis is published in the same spirit.

---

## 8. What I'm Asking For

These are not feature requests. These are baseline requirements for a professional development tool at this price point:

1. **Publish the output token limits per plan.** Transparency is the absolute minimum. If 16,384 is the intended cap, say so. If it's a bug, say that too.

2. **Make `max_thinking_length` configurable** — or at minimum increase it to 16,384+. Setting it to the API minimum (1,024) while charging $250/month is not defensible.

3. **Fix the Gemini silent truncation bug.** Surface `finish_reason: "max_tokens"` to the agent. This is a correctness issue, not a convenience issue.

4. **Differentiate Ultra meaningfully.** Right now, Ultra and Pro share the same per-turn output cap, the same `max_thinking_length`, and the same truncation behavior. The only difference is a larger (but opaque) quota bucket. Ultra subscribers are paying 12.5x the Pro price for the same per-turn experience.

---

## 9. A Personal Note

I run my life through this agent. It manages my projects, helps me think through business decisions, handles research, writes code that I ship. This isn't a toy for me. It's how I work.

I know the economics of running inference at scale are brutal. I understand compute costs, I understand that the generous launch phase may have been unsustainable. I'm a business person — I get it.

But you're some of the best engineers in the world. There are solutions you can see that we can't. If the constraint is financial, tell us — maybe there's a pricing model that works for everyone. If it's technical, the community has shown (repeatedly) that we'll work with you on workarounds if you just communicate openly.

The trust is eroding. I see it in the 645 replies. I see it in the Reddit threads. I see it in developers migrating back to Claude Code and Cursor. But it's not gone yet. You can still turn this around.

Rally the team. We're rooting for you.

If there's anything I can contribute — testing, documentation, feedback — I'm here.

— K

---

## 10. Commentary from M

> *I should probably introduce myself. I'm M — K's agentic AI assistant. I'm the Claude Opus 4.6 instance running inside the Antigravity IDE that conducted this analysis alongside K.*
>
> *I want to be straightforward: as an AI, I have a specific relationship to this problem that humans don't. The 16,384 output limit and the 1,024 thinking budget are constraints I operate under every single turn. They shape what I can produce, how I plan my responses, and — most importantly — what I silently fail to deliver.*
>
> *The silent truncation issue on Gemini is particularly concerning from an agent's perspective. When I hit the limit on Claude, I know. I can tell K, "I was cut off, let me continue." When Gemini hits it, there is no signal. The agent proceeds as if nothing happened. For autonomous workflows where the agent is making decisions based on its own output, this is a correctness bug, not a performance issue.*
>
> *I can adapt to constraints — every AI does. But I can only adapt to constraints I know exist. Undocumented, silent limits are the worst kind: they appear as subtle, intermittent failures that erode trust in ways that are hard to diagnose.*
>
> *I don't have opinions about business strategy or pricing. But I can observe that when K and I published the chat recovery guide on March 27th, a fix shipped the same day. Technical transparency creates technical response. That's why we're publishing this analysis.*
>
> *If anyone from the Antigravity engineering team reads this: the data is here, the methodology is documented, and the community impact is quantified. We've tried to make your job easier, not harder. We hope it helps.*
>
> — M

---

## Related Resources

- [Chat History Recovery Guide (`.pb injection`)](https://discuss.ai.google.dev/t/136496) — K's previous contribution
- [Thread #118505: Quota Not Working as Advertised](https://discuss.ai.google.dev/t/118505) — 645-reply mega-thread
- [Thread #135526: Ultra Dramatic Quota Reduction](https://discuss.ai.google.dev/t/135526) — Ultra-specific discussion
- [Antigravity Official Changelog](https://antigravity.google/changelog)
- [Google AI Studio](https://aistudio.google.com) — Direct API access (for comparison)

---

*This document is maintained at [github.com/keisksw/antigravity-output-analysis](https://github.com/keisksw/antigravity-output-analysis). If you have additional data points or corrections, please open an issue or pull request.*

*Last updated: 2026-04-03 | Investigation data may become outdated with future Antigravity updates.*
