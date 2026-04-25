---
title: "Designing AI Agents for Network Operations: A Three-Layer Harness"
description: "Why the hard problem in applying LLMs to network ops isn't the model — it's the harness."
pubDate: 2026-04-25
---

Over the past several months I've been building an AI agent for my day job in network operations. Triage work that used to take me 15 to 45 minutes now takes 2 to 5. That part is easy to talk about and frankly not very interesting.

What's interesting is everything I had to build *around* the model to make it safe enough to point at production infrastructure. That's what this article is about.

I want to argue two things. First, the hard problem in applying LLMs to network ops isn't the model — it's the harness. Second, the harness has to be designed around a specific asymmetry that most AI agent tutorials ignore, and the design choices follow from that asymmetry in a way that I think is non-obvious.

## Why bother in the first place

Before the design, the motivation. The reason this is worth building, and worth building carefully, is that triage work in network ops has a specific shape that maps unusually well onto what LLMs are good at.

A meaningful fraction of any triage — I'd estimate 60 to 70 percent in my own routine — is information gathering. Pull the alerts. Pull the related tickets. Pull the device state. Pull the path. Pull the recent changes. None of this is hard intellectually. It's just slow, and it's slow in a particular way: the data comes back as JSON over an API, and a human has to read each blob, hold it in working memory, and correlate it with the next blob. LLMs do this kind of structured-text aggregation an order of magnitude faster than humans, and they don't get bored on the eighth ticket of the shift.

The valuable work — judging whether the pattern of evidence implicates the access layer or the transit path, deciding whether to drain a device or wait for more data — is the remaining 30 to 40 percent. That part needs human judgment, or at least judgment that the human can audit. Handing the gathering off to an agent doesn't replace the engineer. It frees the engineer to spend time on the part that actually requires being an engineer.

There's a second reason worth naming. Network ops as currently practiced has a quiet problem with reproducibility. Two engineers handed the same incident will run different procedures, in different orders, with different stopping points. The output of a triage often lives in one engineer's head and gets transcribed into a ticket comment with whatever fidelity that engineer felt like writing. A workflow constrained by an explicit specification and run by an agent produces the same shape every time. The result is something the next shift, the next engineer, or the next post-mortem can actually consume.

These two — speed on the boring part, reproducibility on the output — are why this is worth doing. They are also why doing it badly would be worse than not doing it at all, which brings us to the harness.

## The asymmetry

When an LLM writes code that's wrong, the feedback loop in software engineering is fast and cheap. Compilers reject malformed code in milliseconds. Tests can be re-run thousands of times. Staging environments exist precisely so that wrong code can be observed without consequence. None of these mechanisms are perfect — plenty of bad code still ships — but they exist, they're cheap to use, and the cost of an iteration is measured in seconds.

Network operations does not have this. When an LLM looks at a routing table and concludes the wrong thing, there is no compiler. There is no staging copy of the production network. The feedback loop, if you want to call it that, is the network itself — and by the time the network tells you the conclusion was wrong, you've already shipped a change, drained a device, or paged the wrong team. The cost of an iteration is measured in incidents.

This is not a "networks are special" argument. The same shape shows up anywhere LLMs operate on real-world state without a cheap rollback: industrial control, clinical workflows, financial settlement. Wherever the feedback loops of software engineering aren't available, somebody has to build slower truth gates by hand. That somebody is the engineer designing the harness.

## What the harness actually does

The harness in my system has two parts. There's a piece of infrastructure at the tool layer that lets the rest of the system reason about uncertainty. And there's a control plane at the workflow layer, made of three constraints, that does the actual work of keeping the agent honest. I'll take them in that order, because the second part doesn't make sense without the first.

## Tool layer: making trust a first-class value

The default assumption in most AI agent designs is that a tool either succeeds or fails. The tool returns data, the agent uses the data, life goes on.

This breaks the moment you have tools that fetch real-world state. A tool can succeed in the narrow sense — the API call returned 200, the data is in hand — and still hand back something the agent should not trust. The response was paginated and you only got the first page. The result exceeded the model's context window and got truncated. The data is technically correct but six hours stale because the upstream system is behind.

A tool that returns this kind of result and reports success has not failed. It has lied.

The fix is to refuse the binary. In the agent I'm building, every tool returns its data alongside a small status contract — in my implementation a `_meta` field with three states: `ok`, `failed`, and `untrusted`. The third state is the one that matters. `untrusted` means the tool got data, but the tool itself knows the data is incomplete, partial, or stale, and the agent should treat it accordingly.

Why does this have to live at the tool layer rather than higher up? Because the information needed to make the call is only available at the tool layer. The function fetching ticket details knows whether its response was truncated by a token limit. The function paginating an API knows whether more pages remain. By the time the workflow sees the data, that information is gone. Pushing the trust judgment up the stack means re-deriving it from incomplete signals, which is exactly what `untrusted` exists to prevent.

This is a small change in the type signature. It is a large change in what the agent can do safely, because uncertainty is now a first-class value moving through the system instead of a footnote in someone's prompt.

## Workflow layer: three constraints

With the trust signal available, the workflow layer can do real work. My workflow layer has three pieces, and I want to be specific about why it's three and not two or four.

**The workflow itself** is the action sequence — what the agent does, in what order, to handle a particular kind of incident. This is the part most AI agent designs stop at: a prompt that describes the procedure.

**The workflow specification** is a separate document that declares the workflow's contract: its preconditions, its postconditions, its exit criteria, and the rationale for the ordering. If the workflow is "what to do," the spec is "what has to be true before you start, what has to be true when you're done, and when you're allowed to stop early." The spec exists so that the workflow is verifiable, not just executable.

**The failure mode policy** is shared across every workflow in the system. It says: when a tool returns `untrusted`, here is the decision tree. When a precondition fails, here is what stops and what continues. When two stages disagree, here is which one wins. The policy doesn't have to be elaborate — mine is a few pages — but it has to be external to any individual workflow, so that the agent's behavior under uncertainty is consistent across the system.

The reason this is three layers and not one prompt is that the three pieces answer three different questions. The workflow describes intent. The spec turns intent into a verifiable contract. The policy handles what happens when the contract is broken. Collapse them into one document and you lose the ability to audit any of them independently.

## The loading problem

There is a piece of this that I want to be honest about, because it's a real limitation and pretending it isn't would undermine the rest of the design.

For the three-layer constraint to work, the agent has to actually have the spec and the policy in its context when it executes the workflow. So the workflow itself begins with what I call Step 0: before any tool call associated with the workflow is allowed to run, the agent has to load the spec and the policy into its context and acknowledge that it has done so.

This is not a guarantee that the agent has *internalized* the constraints. LLMs in long contexts exhibit attention degradation — content loaded early in a session gets effectively forgotten as later tokens accumulate. I cannot solve this in the harness. What I can do is move the situation from "the spec was never loaded" to "the spec was loaded but might be partially attended to," and the gap between those two states is much larger than the gap between "loaded" and "perfectly followed."

This is probabilistic engineering, not deterministic engineering. It is one of the things that distinguishes building with LLMs from building traditional software, and I think it deserves to be named rather than papered over. A mechanism that improves the probability of correct behavior is still a mechanism worth having, even if it doesn't deliver certainty. The limitation is real, and it's one of the things the next generation of models — with longer effective contexts and better attention — should partially relieve.

## "Why not just keep it read-only?"

A reasonable reader, especially one who has spent years in network operations, will at this point ask whether all of this is overengineered. If LLMs are uncertain, why not constrain the agent to read-only operations and have a human execute every change? The harness becomes unnecessary, the failure modes become bounded, and we sleep at night.

The argument is reasonable but it misidentifies where the truth gate lives. In a read-only agent, the LLM is not executing changes — but it is producing the diagnostic conclusions that engineers act on. The engineer who reads "the issue is on the access layer based on these symptoms" and proceeds to drain a device is taking action *on the basis of the model's judgment*. The LLM has not touched the network, but its output has. The truth gate isn't the moment of writing. It's the moment of belief.

A read-only agent without a harness is not safer than a write-capable agent with one. It is differently unsafe, in a way that's harder to audit because the human is the one who pulled the trigger and the model's contribution is invisible in the post-mortem.

## The philosophy underneath

The reason I'm writing this as a design philosophy article and not a how-to is that the specific mechanisms — the three states, the read-and-confirm, the policy document — are not the point. The point is the stance.

A single-agent system has a ceiling that is determined by the model. That ceiling rises every time a new model ships and there is nothing the engineer can do about it. The floor, however, is determined by the harness. The floor is what the agent does on its worst day, with bad inputs, in conditions nobody anticipated, on a Friday afternoon when the on-call engineer is the one trusting its output.

Raising the floor is not free. Every constraint the harness imposes is also, in some scenario, a ceiling on what the agent could otherwise have done. An agent that has to load specs before every workflow is slower than one that doesn't. An agent that has to honor a failure mode policy is less flexible than one that improvises. The harness is a deliberate trade — some ceiling, in exchange for a higher floor.

In domains where the floor matters more than the ceiling, this trade is the right one. Network operations is one of those domains. I suspect there are many more.

I have not gone the multi-agent route, in case anyone is wondering. Multi-agent architectures raise the ceiling in some scenarios and almost always lower the floor, because every additional agent is another place a trust signal can get lost in translation. For this domain, the trade is wrong. Yours may be different.

If there's one thing I'd want a network engineer reading this to take away, it's that the engineering work in applying LLMs to ops isn't in the prompt. It isn't in the model choice. It's in the harness — in the specifications, the contracts, the policies, the trust signals. That's where the engineering actually lives, and that's where the next several years of work in this space will have to happen.

The harness I've described is designed for the models we have today. The next generation — with longer effective contexts and better attention, and perhaps the ability to introspect their own state — will make some of these layers unnecessary. Step 0 may stop being needed when models stop forgetting. The `untrusted` state may collapse back into `failed` when models can verify their own tool outputs. But the underlying concept — that an agent operating without natural truth gates needs them designed in by hand — won't go away. It will migrate to whatever the next boundary turns out to be. Harness engineering isn't a transitional discipline. It's a permanent layer of the stack, and figuring out where its boundaries should sit is the work.

---

*Views and opinions expressed here are my own and do not necessarily reflect those of my employer. This article describes general design principles for AI agents in network operations and does not disclose proprietary information about any specific organization, project, or system.*
