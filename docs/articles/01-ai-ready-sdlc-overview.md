# From AI-Curious to AI-Ready: The SDLC Patterns That Separate the Two

*A look at how I'm proving out an AI-driven SDLC end-to-end through CaseFlow AI — a portfolio project written in the open.*

---

Most engineering organizations have adopted AI. Most of them have not adopted an AI-ready SDLC. The difference is becoming the single biggest predictor of who ships well in 2026 and who quietly accumulates technical debt while saying the right things in all-hands meetings.

I've spent the past year leading engineering teams through this transition. The pattern that keeps repeating: a team licenses Cursor or Copilot, declares itself "AI-native," and then runs straight into the same problems that plagued the pre-AI version of the team — only faster, because now bad decisions get codified into sprawling diffs in an afternoon instead of a sprint.

The teams that actually pull ahead are doing something boring and unglamorous: they're treating AI like a new contributor. A very fast, very confident contributor who needs the same scaffolding any new hire needs.

That scaffolding has three legs. I'm building a portfolio project, **CaseFlow AI**, in the open to show what each one looks like in practice. CaseFlow AI is a multi-tenant case-management SaaS — full stack, real infrastructure, real deployment. Every decision is captured. Every architectural reversal is captured. Every sub-phase ships with a 1-page spec written before code is generated.

Here's what I'm proving out, and why each leg matters more than any single piece of code in the repo.

## Leg 1 — Decision logs as first-class engineering artifacts

The most expensive bug I've ever paid for wasn't a bug. It was a decision that nobody on the team could reconstruct six months later. Someone had argued for it in a meeting, someone else had counter-argued, a compromise had been struck, and the compromise lived on as a constraint in the codebase that nobody could justify and nobody could safely remove.

AI makes this worse, not better. An AI pair will happily honor whatever convention exists in the codebase — but it can't tell you *why* the convention exists. If the why is in someone's head and that someone left, the codebase becomes a museum of un-reversible decisions.

The fix is structural. In CaseFlow AI, every architectural decision lives in `/docs/decisions/`, organized by phase. The current Phase 1 (Multi-tenancy + Real Auth) decision log carries:

- **22 pre-flight decisions** — choices made before any code was written. Examples: "JWT secret lives in Secrets Manager, not env vars in plaintext." "Cross-tenant access returns 404, not 403, to avoid existence-leak."
- **8 mid-flight reversals** — choices that were wrong and had to change. Examples: "Reverted `synchronize: false` to `synchronize: true` for dev velocity." "Aligned `@JoinColumn` names with `@Column` property names across four entities after phantom dual columns surfaced."

Reversals are the most valuable entries in the log. They're proof the team was honest about what it didn't know yet, and they form a vaccine against repeating the same mistake under a different name in a future phase.

The format is deliberately lightweight:

```markdown
## Pre-flight 8 — Admin-approval gate

**Decision:** PENDING_APPROVAL users can log in, but data access is
gated by ActiveUserGuard.

**Why:** The originally-proposed alternative (block login outright)
strands the user with no way to learn their account state.
Allowing login + gating data lets the UI show a clear
"waiting for admin approval" page.

**Revised same day** from "block login outright" — that's the kind
of reversal worth capturing.
```

If you can't reconstruct an architectural decision from your repo six months later, you don't have an architecture. You have folklore.

## Leg 2 — Dev-ready specs before any code is generated

The single most predictive question I ask when reviewing an AI-assisted PR: *what spec was this written against?*

If the answer is "the Jira ticket," the diff will be sprawling. Tickets describe outcomes; they don't constrain implementation. The AI fills the gap with its priors — sometimes brilliantly, sometimes by importing the wrong library, sometimes by inventing an abstraction that nobody asked for.

If the answer is "a 1-page spec that names the scope, the acceptance criteria, the files in/out, and the anti-goals," the diff is reviewable in 20 minutes and matches the constraint.

CaseFlow AI sub-phases each ship with a dev-ready spec. The structure:

- **Scope** — exactly what's in, in plain English, with file paths.
- **Acceptance criteria** — testable claims. "User from Tenant A authenticated with a valid JWT cannot read Tenant B's cases." Not "multi-tenancy works."
- **Files in / files out** — which files the AI is allowed to touch, which files it must not. Hard constraint, no exceptions.
- **Anti-goals** — what *not* to do. "Don't refactor `databaseConfig` in this sub-phase." "Don't introduce a new dependency."
- **Resolved pre-flights** — links to the decision log entries that constrain this work.

The anti-goals section is the one most teams skip and the one that does the most work. Without it, the AI optimizes for the diff in front of it and breaks invariants you didn't think to mention.

This isn't process for the sake of process. A 1-page spec takes 20 minutes to write and saves 2 hours of cleanup on the back end of a PR. It also leaves a paper trail that compounds: the spec from sub-phase P1.5 becomes the reference for sub-phase P2.3 three weeks later, and the AI can re-read it instead of re-inventing the convention.

## Leg 3 — Roadmap as a living portfolio artifact

Most roadmaps are theater. A presentation that gets shown at a quarterly review and never opened again. They're optimized for the moment of the show, not for the day-to-day work.

CaseFlow AI's ROADMAP.md is the opposite. It's the first file I open every morning and the last one I update every evening. It tracks:

- **Today's state** — a 2–3 sentence summary of where the project is right now, including dates.
- **Active scope** — what sub-phase is in flight, and what just shipped.
- **Last shipped** — the most recent sub-phase, with a one-paragraph summary specific enough that future-me can reconstruct what happened without re-reading the commit history.
- **Progress bars** — per-phase progress with sub-phase counts. Not a vanity metric — a forcing function to keep sub-phases small enough to actually complete.
- **Cost reminder** — current AWS spend posture. As soon as a stack goes live, the line updates. Keeps me honest about idle cost.

The roadmap doubles as the portfolio surface. A recruiter or hiring manager who clicks into the repo lands on a document that's clearly maintained, clearly current, and clearly the artifact of someone who treats a project like a project — not a demo.

This matters for the audience this series is aimed at. Recruiters in 2026 are no longer impressed by "I built a side project." They're looking for evidence that the candidate operates at senior scale on their own time — not because they have to, but because that's how they think.

## Why this matters for the next two years

I'll make a specific prediction: by 2027, hiring committees will explicitly evaluate candidates on their ability to **lead** AI-assisted delivery, not just use it. The signal will look something like:

- Does the candidate think structurally about how AI fits into the SDLC?
- Can they show artifacts — decision logs, specs, retros — that prove they're doing this on the job?
- Can they tell honest reversal stories without flinching?

That's the bet behind this series. I'm building CaseFlow AI in the open as the artifact that backs the claim, and I'm unpacking each pattern in a separate post over the next few weeks.

Coming up next:

- **Article 02:** *Decision logs as engineering artifacts* — the full format, with real entries from CaseFlow AI's Phase 1 log.
- **Article 03:** *Multi-tenancy isolation patterns* — why cross-tenant reads return 404, not 403, and the latent NOT-NULL bug we caught the second time around.
- **Article 04:** *Real auth without the footguns* — JWT, refresh rotation, httpOnly cookies, and the session bootstrap pattern that survived a deploy.

If you're a recruiter, hiring manager, or engineering leader trying to figure out what AI-native delivery actually looks like — follow along. The repo and the artifacts are public; I'll link them in each post.

---

*Armando Musto is a Technical Lead with a decade of experience shipping enterprise SaaS. CaseFlow AI is his current portfolio project, built in the open at [github link TBD]. He writes about engineering patterns at the intersection of AI and SDLC.*
