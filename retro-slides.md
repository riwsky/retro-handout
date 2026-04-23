# AI-Powered Browser Automation for Insurance Portal Auth

Will Cybriwsky -- project retrospective from Granted, 2024.

Granted is a healthcare-navigation app -- choosing plans, finding doctors, fighting claim denials. To do any of that, we need the user's data, and the only complete source is their insurer's portal. So we log in on their behalf via browser automation.

The talk: why we rebuilt an existing hand-coded system using LLM agents, what we kept from the old one, and what I'd redo.

---

# Context and goals

	**Problem.** Hard-coded selectors covered four big insurers. Expanding beyond them wasn't bottlenecked on dev cost -- it was bottlenecked on account access.

	**Why AI changed the shape of the problem.** An agent can reverse-engineer a novel portal *synchronously, as a user signs in*. Qualitative shift: we can serve the long tail.

	**Goal.** Minimize latency and dev effort; maximize completion rate; support the long tail of insurers.

Pre-LLM, expanding coverage meant fake-enrolling in insurance to reverse-engineer login flows (expensive, legally dicey) or recruiting real users to pair with devs (slow, bad UX). Neither scaled past the head of the distribution.

"Completion rate" rather than "reliability" because users forgetting creds or quitting mid-flow aren't software bugs, but they *are* business impact. That reframe drove product decisions too -- password-reset guides, quit-reason surveys, a dedicated support email.

---

# Technical design: the robustness-speed frontier

	Three approaches on a Pareto frontier:

	- **Configured selectors** -- fast, fragile, covers the head.
	- **Single-shot per-page prompts** -- medium speed, medium robustness.
	- **Full multi-turn agent** -- slow, resilient, handles anything.

	**We shipped all three.** A control loop races them per-page and can switch mid-session.

Key decision: we did *not* start with all three. We started correct -- with the agent -- and added complexity only once we'd measured UX impact on the top portals, where latency mattered most.

The mid-session switching is what makes the hybrid cheap: if configured selectors break on the MFA page, the agent takes over for that step only. We still get the speed boost on everything else.

Anecdote on why the agent can't be fully replaced by per-page prompts: the first portal we saw with separate one-digit-per-input MFA boxes. The single-page prompt contract was locked to "one selector for the code + one for submit." The agent, with full history in context, just made individual fill and click calls.

---

# Technical design: architecture

	Temporal workflow → sign-in engine (control loop + 3 strategies) → Playwright via Browserbase.

	Clients (React Native, Next.js) → NestJS API → Temporal → engine.

	Engine races the three strategies; filtered HTML and screenshots flow back.

Temporal gave us durable execution across user input waits (MFA codes can take minutes). Browserbase handled the anti-bot cat-and-mouse -- residential IPs, fingerprinting -- so we didn't have to.

I also built in-house libraries that are now off-the-shelf: type-safe tool calling (pre-Zod-in-the-OpenAI-SDK), telemetry for LLM calls, retry-aware prompt management. Some of this I'd replace with Stagehand or similar today.

Monitoring: OTel traces in Honeycomb, Playwright traces in S3 (DOM replay for debugging), and a manual review app whose labels later trained an LLM-as-judge to separate reliability failures from user-side incompletions.

---

# Execution

	**Team.** Me (lead eng, frontend + backend of sign-in). Dedicated PM + designer. Two eng teammates on post-sign-in crawl and FHIR ingestion.

	**Rollout.** Daily deploys. Feature flags (LaunchDarkly + our own). Gauntlet of simulated portals in CI.

	**My role.** First person doing agentic LLM work in the codebase -- so also responsible for the shared tooling everyone else used.

Coordination story worth telling: my ingestion teammate started with a YAML DSL for LLMs to write transformations in -- on the theory that a constrained DSL is easier than a real language. But the DSL wasn't in any pretraining corpus, so agents struggled with syntax, let alone semantics. He switched to jq on my suggestion and it immediately worked. Classic jagged-intelligence lesson: "simpler for humans" and "simpler for LLMs" aren't the same axis.

A fun only-in-the-2020s bug: the sign-in agent was occasionally tripping LLM refusals for "adversarially reverse-engineering auth flows" and "handling PHI." We got past it by just explaining the consent and financial-incentive context in-prompt.

---

# Outcomes and reflection

	**Outcome.** Unlocked the long tail before closing the perf gap on the head. Final hybrid: ~30s on big insurers (resilient now), ~45s mid-tier, ≥90s long tail.

	**If I were starting over:**

	- Lean on vendors for anti-bot, sooner. Power-law cuts *against* us here.
	- Let agents run end-to-end sims sooner -- unit tests missed control-flow bugs.
	- Don't pre-invent the wheel. Libraries caught up; some of our infra became redundant.

The before/after table is the one-liner: hardcoded was ~30s and fragile on big insurers, unsupported elsewhere. Initial fully-agentic was ≥90s but resilient everywhere. Final hybrid got us back to ~30s on the head while keeping resilience across the whole distribution.

The anti-bot point is the most interesting reflection. Most of our design leverage came from the insurer-size power law -- the top 20 portals serve the bulk of users, so perf work on them is high-leverage. But that *same* asymmetry means anti-bot work on the top is low-leverage: the big portals have the most sophisticated detection, and we were better off paying a vendor than reinventing residential-proxy rotation.

End-to-end sims: we reserved E2E tests for real portals because setting up simulated ones is expensive in human time. But agents can build those sims *for themselves*, cheaply. The tradeoff shifts once the test-writer isn't a human.
