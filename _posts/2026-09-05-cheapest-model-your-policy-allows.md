---
layout: post
title: "The cheapest model your policy allows"
date: 2026-09-05
excerpt: "Model routing is sold as a cost lever. In a regulated enterprise it's where data policy gets enforced, and the order of those two operations decides everything."
reading_time: 7
---

Every model router I've seen in production optimizes for the same thing: the cheapest model that can do the job.

Almost none of them can answer the first question an auditor asks. Not what it cost. Which models were *allowed* to run this request, and who decided that.

## The piece that started this

Kay Zhu, co-founder and CTO at Genspark, published *The AI Efficiency Frontier* in August. It's the clearest thing I've read this year on why enterprise AI bills explode while model prices fall. The unit price of intelligence drops roughly tenfold a year, but a long-horizon agent burns millions of tokens where a chatbot burned thousands. Price per token goes down. Cost per outcome goes up.

His reframe is the part worth stealing: stop measuring cost per token, measure cost per *successful* task. A cheap model that gets retried, corrected by a human, or escalated to a stronger model was never cheap. It was a slower route to the same bill.

He then walks through how Genspark manages that in production — routing, fine-tuning open-weight models into specialists, and an advisor pattern where a cheap executor consults a stronger model only at the moments that change the outcome, all sitting under a continuous evaluation loop. He calls the emerging discipline AI FinOps. The analogy to cloud FinOps holds better than most analogies do.

Go read it. It's good.

One sentence in it does more work than the rest, and he spends exactly one sentence on it: routing rules can encode not just which model is best for a task, but which models are approved for it, by data sensitivity, geography, or risk tier.

For a product company that's an aside. For anyone shipping AI inside a bank, an insurer, or a healthcare payer, it's the entire architecture.

![Governed model routing — data classification filters the eligible model set before cost and capability routing selects within it](/assets/diagrams/blog/cheapest-model-your-policy-allows-hero.svg)

## The order of operations is the argument

A cost router asks one question: which model is cheapest for this task?

A governed router asks two, and the sequence is not negotiable. First: which models are *eligible* to see this payload? Then, and only then: which of those is cheapest and good enough?

That sounds like a small reordering. It isn't. It changes what the router *is*.

In the first design, the approved-model list is a policy document that lives in Confluence and gets consulted by humans during design review. The router doesn't know it exists. Compliance is enforced by everyone remembering.

In the second, the eligible set is computed at runtime from the request itself, and the router physically cannot select outside it. Compliance is enforced by the system. The policy document becomes configuration.

Security teams have known this pattern for twenty years. It's the difference between a policy and a policy decision point. Nobody grants database access by writing "please only query production if authorized" in a wiki. We built IAM. AI routing is at the wiki stage, and most teams haven't noticed because the cost savings are exciting enough to distract from it.

## What "eligible" actually means

The predicates are more boring and more specific than the phrase "data sensitivity" suggests. In practice a governed router evaluates something close to this on every call:

- **Data classification of the payload**, not the endpoint. A general-purpose summarization endpoint handles public marketing copy on Monday and a member's claim history on Tuesday. Classify the request, not the route.
- **Contractual coverage.** Is there a signed agreement with this provider that covers this data category? A model can be technically excellent, cheap, and completely unusable because legal hasn't papered it.
- **Processing location.** Where does inference physically run, and does that satisfy the residency commitment you made to the customer or the regulator?
- **Retention and training terms.** Zero-retention and no-training-on-inputs are different guarantees, and for some data categories you need both in writing.
- **Version pinning.** A workflow that went through formal validation was validated against a specific model version. Silent upgrades break the validation, not just the outputs.

Any router that can't express those five predicates is a load balancer with good marketing.

## The trade-off nobody prices

Here's the part that gets left out when routing is pitched to an executive as a savings play.

Governance shrinks your eligible pool, and it shrinks it hardest exactly where the tokens are most expensive. For public and internal-only data your eligible set might be a dozen models and the cost engineering is real. For your most sensitive category, the set is often two. Sometimes one. Your savings on that traffic round to nothing.

So the honest pitch is not "routing will cut your inference bill." It's this: routing cuts your bill on the low-sensitivity majority of your traffic, and on the high-sensitivity minority it buys you something you couldn't otherwise have, which is a defensible answer to how a model was chosen.

Sell that to a CFO as cost control and you'll be asked why the savings are concentrated in the workloads nobody was worried about. Sell it to a CISO as an enforcement point and the cost savings arrive anyway, as a side effect.

There's a second cost, and it's the one I find genuinely painful. Kay's architecture treats same-day adoption of new models as a core advantage — a model ships, it enters the evaluation pipeline that day, it graduates into the pool through gradual rollout. In a regulated environment you cannot do that. A new model needs contract review, a privacy assessment, and evaluation against your own task distribution before it's eligible for anything sensitive. Your adoption latency is measured in weeks, and the gap between what the frontier can do and what you're allowed to use is a permanent tax you pay for operating in a regulated industry.

I don't have a clever way around that. Naming it honestly is better than pretending governance is free.

## Where this actually breaks: subagents

Routing at the front door is the easy version. It's also not what modern agent systems do.

An agent that decomposes a task spawns subagents, and each subagent is its own model selection. Kay makes this point about cost — fan-out is only affordable because subagents can run on cheaper models. He's right, and it's the same mechanism that quietly breaks governance.

![Data classification propagating through an agent fan-out, with each subagent inheriting the constraints of its parent's payload](/assets/diagrams/blog/cheapest-model-your-policy-allows-fanout.svg)

The failure is boring and easy to ship. Classification gets attached to the *entry request* rather than to the *payload*. A parent agent cleared for a sensitive workload spawns three subagents, two of them handling only structural work on non-sensitive fragments, one of them handling the actual regulated content. The router sees three cheap subtasks and does what it was built to do.

Classification has to travel with the data through every hop of the execution graph, and every hop needs its own policy evaluation. Which means your policy decision point is called far more often than your architecture diagram suggests, and it needs to be fast enough that nobody is tempted to cache the answer at the wrong granularity.

If you build one thing from this post, build that. Classification as a property of the payload, evaluated at every node.

## The router's log is audit evidence

Kay tracks *router regret* — the share of tasks where a different model would have done better in hindsight — so the router is measured rather than trusted. Good instrumentation, aimed at quality.

The same instrumentation, pointed at a different question, is your audit trail. For each call: the classification, the eligible set, the model chosen, the policy version in effect, and why. That record is what turns "we have controls" into "here is the evidence," and it's the difference between a workflow that clears review and one that lives in pilot for eight months.

Build the log even before you build the sophisticated routing. A dumb router with a complete decision log will pass an audit. A brilliant router with no log won't.

## The call

Build the policy layer first, then the cost layer on top of it.

That's backwards from how these projects usually run, and it feels wasteful, because on day one your eligible set is small and the routing logic has almost nothing to choose between. You'll have built an elaborate mechanism to select between two models.

Do it anyway. Retrofitting policy into a router that was designed to optimize on price means re-deriving classification for every workflow already in production, and you'll be doing it under an audit deadline instead of on your own schedule.

The first version of a governed router I designed got this wrong. The approved-model list was a static config file someone updated quarterly, and classification was inferred from which endpoint the request came through. It worked until endpoints started handling mixed traffic, which took about a month.

Cost engineering is the easy half. It's arithmetic, and the tooling for it is improving fast. The hard half is being able to explain, per request, why a specific model was permitted to see specific data.

If you can't answer that, you don't have a router. You have a very expensive coin flip with good margins.
