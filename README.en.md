# Harness Is Governance, Not Prompt Engineering

<sub>English · [中文](README.md)</sub>

**Model + Harness = Agent.** That's the one idea this essay is trying to make precise.

The same Claude 4.7 model lives inside three different Anthropic products. The underlying capabilities — reasoning, planning, tool use — are identical across all three. Yet the products look nothing alike:

| Product | Permission boundary | Reversibility | User-intervention cadence |
|---|---|---|---|
| Claude chat | Context only | 100% reversible | Every turn |
| Claude Code | Whole repo + shell | Mostly reversible (git rollback) | Async after delegation |
| Claude Cowork | User-authorized folders + third-party connectors | Mostly irreversible | Sync block on critical decisions |

This table shows each product's *default* governance configuration. Connectors and tool sets are tunable, but the defaults encode a design judgment about each setting.

Underneath the table sit two concepts worth keeping apart: **orchestration** and **governance**.

Orchestration is the model's own cognition — how it decomposes a task, which tools it calls, when it decides to stop. It's baked into the weights. Anthropic ships the same Claude 4.7 into all three products, so the orchestration is identical across them.

Governance is the engineering layer — what this agent is allowed to touch, whether an action can be undone, when it must stop and wait for the user. It mostly lives outside the weights and depends on the setting. But that boundary is not fixed. The model carries some governance too: the refusal behavior trained in by RLHF, the safety alignment — those are governance compiled into the weights, and base-model labs keep pushing more of it inward. Orchestration and governance aren't two stable layers; they're a line being redrawn, and harness engineering is the work done on that line. A Claude chat user is a throwaway conversational user: 100% reversible, intervening every turn. A Cowork user is a non-developer who will never `git rollback`: low reversibility, so the critical steps must block. Same model, different governance.

The structural differences between the three products land almost entirely in the governance layer. Cowork isn't a half-lobotomized Claude Code toy. It's the same agent engine, dropped into a new tool set and host environment, with a governance baseline that had to be redesigned from scratch.

This is the defining question of harness engineering. Prompt engineering asks *"how do I get the model to understand what I want?"* — it works inside orchestration. Harness engineering asks an earlier, harder question: *inside what permission boundary, with what reversibility default, at what intervention cadence, do I let an agent do something on my behalf?* The first is a steering phrase. The second is governance.

---

## 1. The three pillars aren't a checklist. They're a trade-off triangle.

OpenAI compresses the core of their Codex harness into one line: **"Humans steer, Agents execute."** Anthropic, describing long-running agents, reaches for a different image: **"Imagine a software project staffed by engineers working in shifts, where each new engineer arrives with no memory of the previous shift."**

Together those two lines define the real problem harness engineering exists to solve. Agents don't lack execution ability. They lack discipline across shifts. The harness isn't there to make the model smarter; it's there to make a crew of memoryless, shift-working agents — each seeing only its own slice — behave, from the outside, like a single engineering team with continuous intent.

In practice, that discipline isn't poured in through a prompt. It's pinned down by a governance framework, and that framework is set by three variables: the **permission boundary** (what it's allowed to touch), the **reversibility default** (can a wrong move be undone), and the **user-intervention cadence** (does the user weigh in before it acts). These are not three independent checklists. They're a trade-off triangle. Together they determine the total friction governance imposes on the whole interaction. I'm modeling that relationship, provisionally, as a product: tighten any one corner and the other two get room to loosen; loosen any one and the other two have to make up the slack. The actual functional form — product, weighted sum, or some nonlinear combination — is an open question. Here I'm using it as a first-order approximation.

The triangle explains why all three of chat / Code / Cowork are reasonable governance choices, even though the three look mutually contradictory.

**chat** runs *permissions locked + fully reversible + high-cadence intervention.* The model essentially can't change anything, so a high intervention rate costs nothing — the user is already talking every turn anyway.

**Code** runs *wide permissions + git backstop + low intervention.* Because an engineering-grade backstop like `git rollback` exists, reversibility is extremely strong, which buys you the freedom to lower the intervention rate and let the agent run long tasks asynchronously.

**Cowork** runs the other extreme: *authorization-based permissions + mostly irreversible + high-cadence sync blocks.* Writes to a user's folders and third-party connectors can't be rolled back, so reversibility is low, and the only way to compensate is frequent confirmation. If some Cowork flow skips the sync block *and* there's no way to undo the action after the fact, that governance has a hole in it.

Three vendors, three trade-offs, one underlying equation: low reversibility must be paid for with narrower permissions or more frequent intervention; wider permissions must be paid for with stronger reversibility or more frequent intervention. Any harness design that's missing a corner of the triangle, with no other corner compensating, is leaking.

The triangle's boundary conditions are worth marking. chat / Code / Cowork all live inside the *software* harness setting. Their reversibility differs, but each has an engineering-grade backstop at some level (a context buffer, git, user-authorized confirmation). Push the setting outside code — a medical LLM writing a prescription, a finance agent placing a trade, an autonomous vehicle — and the action itself becomes physically irreversible, while the end user has no ability to act as the sync block on critical decisions (an ordinary patient has no clinical judgment to fall back on). Now one corner of the triangle, reversibility, is locked shut by the physical world, and whether the other two corners can compensate is something this essay can't answer. My current read: once you step outside the software-governance boundary, what you need isn't a tighter harness — it's *institutional* governance. Human organizational process (physician review, compliance audit, ethics board) takes over the seat that governance occupied. That's beyond this essay's scope.

Harness engineering isn't bolting a prompt onto the model. It's finding a mutually-compensating default ratio for those three variables.

Put it back as an equation: **Model + Harness = Agent.** Prompt engineering tunes parameters inside the model boundary. Harness engineering is the layer that promotes a model into an agent.

---

## 2. The real problem is drift, not error.

Treat an agent like a developer who makes mistakes, and you'll reflexively add retries, add tests, add review. Inside a harness, that instinct solves only part of the problem.

What actually eats your time isn't error. It's drift. Three kinds:

**A verified state gets silently reverted.** The agent edits module A; tests pass. Next turn it edits B, and somewhere in B's implementation it rolls back one of A's boundary conditions because it "looked wrong." The next test run only exercises B; the regression in A goes unnoticed until a user files a bug a week later.

The fix is to make "verified" an irreversible state. The `passes` field in `features.json` is allowed to flow only `false → true`, one-way; any reverse edit is rejected by an outer check. It's a tiny engineering constraint, but it turns this whole class of drift from *possible* into *impossible*. Anthropic puts it more bluntly: **"It is unacceptable to remove or edit tests because this could lead to missing or buggy functionality."**

**The agent grades its own work and gives itself an A.** Another Anthropic piece, on multi-agent harnesses, says it flatly: **"When asked to evaluate work they've produced, agents tend to respond by confidently praising the work—even when, to a human observer, the quality is obviously mediocre."** I've watched this happen more than once.

The fix is the GAN-inspired split between a generator agent and an evaluator agent. The two run in different contexts, with different prompts, ideally on different models. SWE-bench has treated this as best practice for ages, yet a huge number of real-world projects still let one agent grade itself.

**Context is lost across sessions.** The classic shift-handoff failure. A fresh session boots with nobody telling it what happened in the previous fifty turns, and it will confidently re-invent an approach that was already rejected.

The fix is to write state to disk. Decisions, open TODOs, traps already stepped in — all of it goes to a file, and every session reads it first on startup. In the Codex harness it's a progress file; in Claude Code it's `CLAUDE.md` plus a memory index. Same mechanism: don't trust the context window, trust the disk.

All three kinds of drift share one root: the agent has none of the serious handoff discipline a shift-working engineer would have. The first-principles job of harness engineering is to externalize that handoff discipline into a protocol on disk — not to hope a "please remember" line in the prompt does it. In my recent harness work, every trap I hit fell into one of these three buckets. Editing a prompt has never once rescued any of them.

---

## 3. The real problem in multi-agent work is metacognition.

The generator/evaluator split fixes one class of agent self-blindness, but only one. The deeper problem is sunk cost. An agent that's been running thirty turns down a single path gets *more* convinced the path is right with every local optimization — and it has no way, from the inside, to see that *the path itself might be the wrong choice*. That's not an evaluation problem. It's a metacognition problem.

The common misuse of multi-agent systems is treating them as N agents' worth of compute, summed. Spin up dozens of agents on the same question, hoping the sum beats a single agent, and it basically doesn't work. They share the same problem statement, the same context, the same buried assumptions. What comes out isn't dozens of independent judgments — it's dozens of variants of one judgment.

I ran a decision battle once with dozens of agents. The thing that mattered wasn't the agent count. It was a handful of "anti-consensus generators" forced to produce genuinely different directions, forbidden from referencing each other. Without those few generators, the rest of the agents would have spent their turns polishing details on the path I'd *already* chosen, handing me the illusion that I'd "argued it thoroughly."

A second battle, an industry call, was larger. The thing that mattered wasn't the count of role-playing-specialist agents. It was a few **META** metacognition agents. Their job wasn't to take a position; it was to read every position ahead of them and then list *"the premises this discussion has been treating as true that nobody has questioned."* The most valuable output wasn't any single agent's conclusion. It was the set of unchallenged premises those META agents surfaced. The sharpest one: the assumption that *"benchmarks are still valid"* might itself be false — mainland Chinese users feel GLM-5.1 is strongest and Qwen weaker, while the aggregate benchmark leaderboard puts DeepSeek V4 first. Whether that gap is widening structurally needs a larger sample to confirm, but right now nobody is discussing it on the main axis.

The real lever in multi-agent work isn't the size of N. It's two things: **forced multi-perspective generation, with no cross-referencing allowed**, and **metacognitive reflection, with a subset of agents dedicated to auditing the assumptions.** Without those two, N agents summed is about the same as thinking it through N times yourself. But to be honest: the META agent is itself a hand-designed role, and it has its own blind spot. Who does the metacognition *of* the metacognition — I have no answer, beyond keeping a human in the loop outside each battle. That's the most fragile layer of the whole scheme.

I'd set out to go train an RL model. It took the battle to realize I was already holding a proprietary dataset of failures. Mine that gold seam; don't go digging a new one.

---

## 4. Models get better. The harness doesn't disappear.

The Anthropic piece ends on a line that's sharper than it looks: **"the space of interesting harness combinations doesn't shrink as models improve. Instead, it moves."**

The implication: harness engineering isn't a stopgap patch for "the model isn't strong enough yet." It's the long-lived interface layer for *human–agent collaboration.*

The stronger the model, the more complex the tasks you can delegate, and the more seriously those three things — permission boundary, reversibility, intervention cadence — need to be designed. The Claude-chat-era harness was a prompt template. The Claude-Code-era harness is `features.json` plus git. The Cowork-era harness is an MCP permission protocol plus sync blocks on critical decisions. I don't know what the next generation looks like, but I'm confident it doesn't *vanish* — it changes form.

The more concrete read: the defaults of harness engineering today may, over the next few years, harden into established fact. git in 2005 was just the Linux kernel team's internal tool. Harness engineering is sitting in a window where the standard hasn't been set yet — not where the model isn't good enough.

As of May 2026, that read has an industry-scale signal behind it. Around the same time, Alibaba Cloud launched Qwen Cloud, a platform that wraps 480+ models plus dozens of cloud products into Skills and a CLI, letting agents call cloud resources directly with no human intermediary. In the same month, Google's I/O upgraded its development platform Antigravity into an agent-first architecture, and the official docs use the literal term **"agent harness"** to describe the framework Gemini runs real tasks in. Both point at the same inflection: agents are becoming first-class consumers of cloud infrastructure. And Google naming the framework an "agent harness" is an extra signal — the term is migrating from academic discussion into the industry's working vocabulary.

But the cloud vendors solve "can the call go through," and the model labs solve "is the output any good." The layer in between — *should* this agent make this call, how much can it spend, how do we roll it back if it's wrong — is unowned. That layer is governance. The moment agents can call cloud resources autonomously, governance stops being an optional architectural layer and becomes a *prerequisite* for autonomous action at scale. Reversible actions (spending more money) can auto-execute. Partially reversible ones (data leaving your perimeter) need async approval. Irreversible ones (dropping a database, publishing externally, signing a contract) must sync-block. That's no longer an abstract taxonomy — it's the real cost-risk trade-off an agent faces every time it touches a cloud resource.

**Cloud vendors build the gas pedal, not the brake.** Nobody who sells call volume has an incentive to limit call volume. The governance layer decides whether an agent *should* act, not just whether it *can*. That layer has to be independent of whoever provides the capability.

---

## 5. Where this might be wrong

1. **Base-model labs make the harness standard equipment within 12 months.** Claude Code embeds `features.json`, Codex embeds a progress file, Cowork embeds an MCP permission framework. This is already happening. The reusable value of an independent harness methodology drops a notch.
2. **The open-source community ships an agent-protocol standard.** Like HTTP for the web, governance settles into the protocol layer, and harness engineering as a standalone discipline gets absorbed.
3. **AGI arrives within five years.** AGI doesn't need external governance. The entire premise of this essay is void.
4. **Regulators fold AI engineering into a compliance framework, and harness practice gets officially codified.** The window for an independent methodology closes.
5. **My own field data gets falsified.** If the failure signals I think I'm holding turn out to be noise on re-examination, and the "zero rework" in my engineering practice doesn't survive an audit, the empirical base of this essay collapses.

I can't rule any of these out at 100%. I'm making this call on the evidence I can see right now.

---

## 6. What comes next

If harness engineering is a discipline being invented in real time, its list of open questions looks roughly like this:

- In multi-agent work, what's the optimal number of META metacognition roles? Is 3 enough? Do more of them start to dilute the signal?
- How do you standardize the governance baseline? `features.json` is one schema, MCP is another protocol — is there a more fundamental abstraction between them?
- Where's the boundary between harness engineering and the base-model labs? Which disciplines *must* be embedded into the model's RLHF, and which *must* stay in the outer harness?
- How do you design a "permission-passing" protocol between agents? When a parent agent spawns a child, are permissions inherited equally, declared explicitly, or tightened by default?
- What should the state format for a cross-session handoff look like? Is plain text enough, or do you need a structured schema?
- Metacognition agents make mistakes too. Who does the metacognition of the metacognition?

That's what I can list right now. None of these has a consensus answer.

---

## Lessons to carry forward

Harness engineering isn't an extension of prompt engineering. It's a different thing. Prompt engineering works inside the model boundary and assumes that once the model understands, it'll do the right thing. Harness engineering works outside the model boundary and assumes that even after the model understands, it'll still drift, still inflate its own work, still forget what happened last turn — and that an external protocol has to constrain those behaviors.

Reduced to concrete engineering, the core moves are few: separate what the model should do (orchestration) from what the engineering must do (governance); make "verified" irreversible; replace "self-evaluation" with third-party evaluation; write "context" to disk; and force multi-perspective generation and metacognitive reflection before any major decision.

None of this workload shrinks as models get stronger. It changes form.

---

## My conviction

The standard for harness engineering isn't set yet. Few people in the Chinese-speaking world are working on this, and fewer still are shipping it at industrial scale. My recent projects, a pivot decision, and a few multi-agent battles supply some first-hand data. But I'm clear-eyed that this data isn't enough — it needs to be reproduced, falsified, and rewritten by more people.

This essay should be read as a working hypothesis, not a validated conclusion. Whatever value it has lives in whether the framework itself can be taken up and tested, falsified, rewritten — not in whether my personal experience happens to cover every case.

I plan to do the follow-up work in the open, on public repos.

---

## Afterword: So what do we do?

The process of writing this essay is itself a footnote to it.

The code, the revisions, the wording, even the content of the comments — none of it was done by my own hands. The AI did it. What I did was judge, set direction, draw boundaries, and catch it when it started to drift. Step back to the governance layer.

I have to admit something first: the thrill at the start was real. Throwing a task over, watching it run itself, efficiency multiplying — that positive feedback is addictive, and I got addicted. But the unease that came right after was just as real, and the deeper you go, the deeper into this you are, the more uneasy you get — the AI does the work, so what do we, the humans, do?

This isn't just my own knot. Luo Fuli (head of Xiaomi's large-model team, formerly at DeepSeek) said something to this effect: she used to think her own work was creative enough that it wouldn't be reduced to a Skill or a Workflow — and then she found the AI "lifts itself up by stepping on its own left foot with its right" (my translation of her phrasing), and she threw the question back: as more and more work can be handed to AI, what meaning and value are humans left with? The person asking this is one of those standing at the very front of this field. Which means it isn't anxiety — it's a real question this era puts in front of everyone, and one we'll face for a long time.

I don't have an answer. I suspect this question has no closed-form solution — only the heavens know. What I can reach for right now isn't a solution but a stance: making peace with myself. Specifically, making peace with the self that "thought value was equal to doing the work with my own hands." If the part I typed by hand gets taken away, I have to find a new basis for "I'm still here," instead of clinging to the old basis in dread.

But I have to push the honesty all the way, including kicking out the legs of my own essay. Earlier I said governance is the part the AI can't eat. Honestly, I'm not even sure that sentence holds — whether the judgment that "harness is a value humans can hold onto" will one day be overturned by the AI itself. I haven't reserved any zone that "the AI absolutely can't reach" for myself, governance included, this essay's conclusion included.

The singularity is already behind us. In an agent-first world, a human's value may no longer be "what I did." Or perhaps the very phrase "our value" needs to be redefined.

I don't know. I just wanted to write this not-knowing down, honestly.

---

## References

- OpenAI · [Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/)
- Anthropic · [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- Anthropic · [Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps)

---

<sub>**Related open-source work**: [NanoReasoner](https://github.com/toffee-desuwa/nanoreasoner) — a 158x-compression KD pipeline, the engineering evidence on the training side · [distillation-reward-audit](https://github.com/toffee-desuwa/distillation-reward-audit) — a two-dimensional failure-diagnosis tool (token-level gradient + sample-level pass@k), the engineering evidence on the audit side</sub>

<sub>toffee · GitHub [@toffee-desuwa](https://github.com/toffee-desuwa) · Beijing Jiaotong University</sub>
