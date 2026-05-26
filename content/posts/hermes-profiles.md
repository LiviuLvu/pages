---
title: "Hermes Profiles - Running multiple agents with different personality"
date: "2026-05-26"
summary: "This is a short discussion between me and the LLM about Hermes profiles. Initally it sounded like a cool idea, then I started to think in practice about implementation and it started to get fuzzy."
# description: ""
tags: ["Home Lab", "Second brain", "Agents"]
author: "Liviu Iancu"
weight: 3
series: ["Automation"]
# categories: [""]
# aliases: [""]
# cover:
#   image: images/img.png
#   caption: "Description [text](https://link.somewhere/)"
---

This is a short discussion between me and the LLM about Hermes profiles. Initally it sounded like a cool idea, then I started to think in practice about implementation and it started to get fuzzy.

Conclusions at the end.

---
## *Is this worth the effort? Different Hermes profiles (coder, researcher, business, QA).*

### *https://hermes-agent.nousresearch.com/docs/user-guide/profiles*


Yes, but only if you are already hitting real friction with a single-agent setup.

For your workflow specifically (coding + infra + QA + business exploration + research), profiles are one of the cleaner Hermes features because they isolate:

- memory
- SOUL.md behavior
- sessions/history
- tools/config
- API keys/models
- gateway state  
    without needing separate installs.

The main value is reducing context contamination.

Example:

- `coder` profile remembers coding conventions, project structure, infra commands
- `researcher` profile stays verbose, source-heavy, exploratory
- `qa` profile becomes adversarial and checklist-driven
- `business` profile avoids technical rabbit holes and focuses on MVP/risk/pricing

That separation matters more once memories and long-running sessions become large.

For your use case, these are probably worth it:

- `coder`
- `researcher`
- maybe `qa`

I would skip `business` initially. It tends to become low-value overhead unless you use it daily.

The hidden downside is operational complexity:

- duplicated configs
- multiple SOUL.md files drifting apart
- remembering which profile you are in
- duplicated tool setup
- fragmented memory
- more maintenance during upgrades
- harder debugging when behavior changes

You can accidentally over-specialize agents and lose cross-domain reasoning. A “coder” profile may stop thinking about product constraints; a “business” profile may lose technical realism.

Another important point:  
profiles are not sandboxing. Hermes explicitly states profiles do not isolate filesystem access.

For many solo developers, a better progression is:

1. Single profile
2. Strong `SOUL.md`
3. Better project memory structure
4. Then add 1 specialized profile only when pain appears

Do not create 5+ profiles early. Most people create organizational debt faster than productivity gains.

Docs:

- [Hermes Profiles Guide](https://hermes-agent.nousresearch.com/docs/user-guide/profiles/?utm_source=chatgpt.com)
- [Profile Commands Reference](https://hermes-agent.nousresearch.com/docs/reference/profile-commands/?utm_source=chatgpt.com)

---

## *Give 3 examples of usecases from real users if you can find any*


Three realistic profile-based setups people are actually using or describing publicly:

### 1. Researcher → Writer → Ops pipeline

A user described running four Hermes profiles:

- `researcher`: scrapes Reddit, GitHub releases, competitor changelogs
- `writer`: converts notes into newsletter drafts
- `coder`: debugs scripts and automations
- `ops`: summarizes Slack + Jira into daily digests

The important part is each profile had different memory, tools, and model behavior. The researcher stayed exploratory while ops stayed concise and scheduled.

This is probably the closest to your RSS/research/home-lab interests.

### 2. Always-on “background operator”

One user ran Hermes continuously with:

- Google Calendar
- Gmail
- Todoist
- Obsidian
- Telegram/iMessage

Their workflow chained multiple agents:

- one profile gathers inputs
- another analyzes priorities
- another delegates coding tasks
- another updates notes/memory

The profile system mattered because each agent maintained independent identity and memory instead of becoming one overloaded mega-agent.

This aligns strongly with:

- your markdown/wiki memory ideas
- automated infra summaries
- daily operational monitoring

### 3. Parallel engineering team simulation

A team described running many Hermes instances in parallel:

- backend monitoring
- benchmark creation
- RL/training environment tasks
- code workflows

Another builder mentioned “sub-agent spawning with 5 profiles for delegation and parallel work,” including DevOps/security/research specializations.

This is where profiles become genuinely useful:

- separate API budgets
- isolated prompts
- isolated tool permissions
- independent task queues

Without profiles, long-running multi-agent systems tend to collapse into prompt pollution and memory contamination.

Common pattern across nearly all real users:

- one general profile becomes chaotic over time
- profiles are mostly about reducing cognitive/context interference
- the best results come from long-running background workflows, not interactive chat

Also worth noting:  
many experienced users eventually reduce the number of profiles after experimentation. A common stable setup seems to become:

- `main`
- `research`
- `coder`  
    instead of 10+ “employee” agents.

--- 

## *I'm not convinced this is useful. My main concerns are:*
**1. observability – how can I see what sub agents are doing?**  
**2. Increased token consumption?**  
**3. My mental context overhead keeping track of what agent does what on which project?**  
**4. If I have a QA agent and a coding agent, what stops them from continuously going back-and-forth?**

---

Your concerns are the same reasons many people abandon multi-agent setups after the initial excitement phase.

For a solo developer, most “agent teams” become organizational theater unless there is a strong operational boundary.

Your four concerns are valid:

1. Observability
2. Token explosion
3. Mental overhead
4. Infinite coordination loops

Those are the actual hard problems, not prompt engineering.

## 1. Observability is weak in most agent systems

This is the biggest issue.

Most frameworks show:

- final answer
- maybe traces
- maybe logs

But not:

- why the agent delegated
- what context it received
- whether it misunderstood requirements
- how much hidden reasoning/token burn occurred
- whether the sub-agent hallucinated intermediate state

You end up debugging an organization instead of debugging code.

In practice, experienced users usually end up forcing:

- explicit task contracts
- append-only logs
- artifact outputs
- strict iteration caps
- single-line summaries after each delegation

Otherwise you get “silent chaos.”

Your earlier idea about centralized log aggregation for Hermes is actually more mature than most agent setups people run.

## 2. Token consumption grows nonlinearly

The hidden issue is duplicated context.

Example:

- main agent sends 20k context to QA
- QA sends back 10k findings
- coder re-reads all findings
- researcher gets partial history again

Now every loop multiplies:

- context window usage
- summarization costs
- memory writes
- tool calls

A 1-agent task can become 5–15x more expensive.

This gets especially bad with:

- long memories
- wiki systems
- conversation replay
- “reflection” prompts
- recursive delegation

Most production systems eventually introduce:

- aggressive summarization
- ephemeral agents
- stateless workers
- strict context pruning

Otherwise costs drift upward forever.

## 3. Mental overhead is real

This is the underrated problem.

You start needing to remember:

- which profile owns which memory
- which model works best for which task
- which agent has filesystem permissions
- which agent touched which branch
- which agent already analyzed something

At some point the system stops reducing cognition and starts consuming it.

For solo developers, this is often the collapse point.

The winning systems are usually:

- 1 main agent
- 1 specialized research agent
- maybe 1 executor/QA worker

Not “virtual companies.”

## 4. QA ↔ coder infinite loops absolutely happen

This is extremely common.

Pattern:

- QA finds issue
- coder patches
- QA invents new edge case
- coder refactors
- QA expands scope
- repeat forever

This happens because neither agent has:

- economic constraints
- business priorities
- stopping conditions
- practical tradeoff judgment

Humans naturally stop because:

- deadline
- cost
- “good enough”
- product priorities

Agents do not naturally converge.

The fix is architectural, not prompting.

You need:

- explicit acceptance criteria
- max review passes
- severity thresholds
- termination rules

Example:

```
qa:  max_review_cycles: 2  allowed_severity:    - critical    - high  ignore:    - stylistic issues    - hypothetical edge cases
```

Without hard constraints, multi-agent systems tend toward endless optimization loops.


For your workflow specifically, I think the highest ROI setup is probably:

```
Main Hermes agent├── occasional Research worker└── occasional QA worker
```

Not persistent autonomous teams.

The sub-agents should ideally:

- be stateless
- receive tiny scoped tasks
- produce artifacts only
- die immediately afterward

Example:

```
"Analyze these logs and output only:- critical issues- likely root cause- confidence score- suggested next command"
```

---

## Conclusion

I prefer my automations in my home lab to be highly deterministic, using scripts, having simple setup as possible.

Usually working on projects involves isolated tasks, short lived, fast feedback, continuous steering the AI to not jump overboard with code changes, user flows, rate limiting, security concerns and other.

My conclusion is I dont need Hermes Profiles yet.