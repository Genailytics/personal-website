---
layout: post
title: "Context engineering is just RAG with a compliance layer"
date: 2026-05-23
excerpt: "Everyone's declaring RAG dead and context engineering the winner. For regulated teams that's the wrong fight — the retrieval mechanism was never the hard part."
reading_time: 7
---

"RAG is dead, context engineering won."

It's the take of the month. Budget data backs it up — retrieval optimization spend just overtook evaluation spend for the first time. The framing is that we've stopped stuffing chunks into a pipeline and started letting the model pull what it needs.

Half of that is right. The wrong half is the half that matters if you ship into a regulated industry.

## The rebrand hides the actual decision

Strip the marketing and "context engineering" is an umbrella term. RAG is still in there — embedded queries, similarity search, top-k injection. What changed is that retrieval is now one tool among several for deciding what the model sees, instead of the whole strategy. Long context windows handle some of it. Structured business data handles some. Memory handles some. Fine.

But that's a capability story, not an architecture decision. The architecture decision is which of three things you're actually building:

![Three retrieval-and-governance approaches and what each one sacrifices](/assets/diagrams/blog/context-engineering-compliance-layer-hero.svg)

Plain RAG retrieves top-k chunks. Fast, cheap, and opaque — you can't easily say why a chunk showed up or whether the user was allowed to see it. Full context engineering assembles the whole working set the model needs for the task. Richer, more flexible, and you pay for it in tokens and latency. Governed context engineering does all of that and adds the part nobody puts on a slide: it decides what's eligible *before* anything gets retrieved, and it records what happened.

Most "RAG vs context engineering" posts stop at the first two. For a consumer chatbot, that's the whole conversation. For anything that touches regulated data, the third column is the only one that survives contact with a compliance review.

## The part the demos skip

Here's the trap I've watched teams walk into. The retrieval-then-filter order feels natural: pull the relevant chunks, then redact or filter whatever the user shouldn't see before you respond.

![Filtering after retrieval leaks; scoping by permission before retrieval doesn't](/assets/diagrams/blog/context-engineering-compliance-layer-2.svg)

By the time you're filtering the output, the restricted passage has already entered the prompt. The model has already conditioned on it. You're not preventing a leak, you're hoping to catch one. In a domain with row-level access rules — who can see which document, which clause, which customer's record — that's not a bug you patch later. It's the wrong order of operations baked into the foundation.

Governed context engineering flips it. Permission scoping runs first, retrieval runs inside that scope, and the model only ever sees what this specific user is allowed to see. Same components, inverted order, completely different audit posture. The directional flip is the whole point — and it's the part that's invisible in a demo, because demos run as one all-powerful user.

## Reproducibility is the feature, not the metric

The deeper reason regulated teams can't treat this as a rebrand: an answer is only as defensible as your ability to recreate it.

Say a model gave a wrong classification last Tuesday and someone's now asking why. With chunk-based RAG, you usually can't reconstruct the exact context that produced it. The index has been re-embedded since. The documents changed. The retrieval was non-deterministic. You're left explaining a decision you can't reproduce, which in a compliance setting is indistinguishable from having no answer at all.

A governed context system treats the assembled context as a recorded artifact. You can replay precisely what the model saw at that moment — same documents, same permissions, same ordering. That single requirement reshapes the whole stack. It's why you log the context, not just the output. It's why retrieval order is deterministic where it can be. It's why "which chunks, for which user, at what time" is a first-class field and not an afterthought.

I ran a head-to-head retrieval evaluation on 300-page compliance documents — chunked RAG against a structure-aware approach — and the accuracy numbers were close enough to argue about. What wasn't close was explainability. One approach could tell me *why* a passage was retrieved and tie it back to a section of the source. The other gave me a similarity score and a shrug. For a workflow that has to defend its outputs to an auditor, that gap decided the architecture before precision@k ever entered the conversation.

## The call I'd actually make

If you're building for a static corpus, a single agent, and no compliance pressure: use plain RAG and move on. Don't let anyone talk you into a context-graph cathedral for a FAQ bot. The simplest thing that retrieves well is the right thing.

If you're in a regulated domain — telecom, healthcare, pharma, finance, anything audited — start from governed context engineering and treat the governance layer as load-bearing, not a wrapper you add at the end. Build the permission filter before the retriever. Log the assembled context as an artifact. Make reproducibility a design constraint from day one, because retrofitting it is brutal and usually means a rebuild.

What you sacrifice is real: setup time, more moving parts, a slower path to the first impressive demo. The governed path looks worse in week one and decisively better the first time someone challenges an answer. That's the trade — you're spending early velocity to buy the ability to stand behind your system later.

The mistake I see is teams picking based on the demo, where governance is invisible and the lean pipeline always looks faster. Then the first security review lands and the project stalls — not because the model was wrong, but because the layer around the model can't answer the questions a regulator asks.

The retrieval mechanism was never the hard part. It hasn't been for a while. The hard part is proving, after the fact, that the system only ever saw what it was allowed to see and can show its work. Call that context engineering if you want. In regulated AI it has an older name: doing it properly.

So before you join the "RAG is dead" chorus, ask the only question that ends the debate — if one of your answers gets challenged six months from now, can you recreate exactly what the model saw? If the answer is no, you didn't skip RAG. You skipped the part that mattered.
