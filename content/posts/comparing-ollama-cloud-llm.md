# Ollama Cloud Models Review

**I Switched from Claude Code CLI monthly subscription to Ollama** for about a month to try Hermes and Opencode. The main selling point for me was the memory management Hermes claimed to provide.

**Claude big downside** was how fast I kept bumping in the subscription limit. The main limitation was at the time that I could not use any other harness except the one from Anthropic.

**Overall, I'm pretty happy with Ollama cloud models + Hermes** considering the cost and the amount of output I managed to get while working on my home lab and personal projects, mainly rss-insight, an app I personally use to filter news and reduce doom scrolling.

**There are downsides** too. In this post I highlight some observations from the cloud models I tried using the minimum Ollama subscription.

## General observations

Across most models I tested, I noticed a few recurring patterns:

* They often fail to use `pnpm` already present in the project and default to `npm`.
* They need constant supervision. Strange things happen when they get close to their context limit or after summarization.
* If an instruction is misinterpreted, they usually run with it instead of asking clarifying questions.
* The same model can contradict itself, sometimes within the same response.
* Models frequently find mistakes in code or output they generated themselves earlier.
* Having one model review another model's output often reveals problems.
* Complete flop during a UI library migration, even with guidance. The result technically worked but the UI was practically unusable.

## Models that worked reasonably well

### kimi-k2.6:cloud

**+** Easy to steer once I got used to it.

I used it almost exclusively for a month before comparing it with other Ollama Cloud models. After trying alternatives, some drawbacks became more obvious.

**-** Becomes very unpredictable when it reaches about half its context, aprox 200K tokens.

**-** Ignores previous instructions, such as holding upgrades until a migration is complete.

**-** Can contradict itself in the same prompt.

**-** When asked to reduce duplicate items, sometimes reintroduces items removed earlier.

### deepseek-v4-pro:cloud

**+** Produces very structured responses and follows `agents.md` well.

**-** Occasionally reports tasks as completed while leaving the project in a broken state.

## Models I couldn't really get along with

### minimax-m3:cloud

**-** Extremely verbose.

**-** Difficult to review and verify.

**-** Rarely asks useful follow-up questions.

**-** Slightly too sycophantic for my taste.

### glm-5.1:cloud

**+** Fast responses.

**+** Generally follows `agents.md`.

**-** Sometimes reports tasks as complete while the project is still broken.

**-** Occasionally ignores previous instructions.

**-** Stops unexpectedly after compacting and requires prompting to continue.

**-** Failed badly on straightforward instruction where others got right:  
  - Asked to create a file using its own name. It used the harness name instead.  
  - Unacceptable: instead of creating the file locally at the path specified in the prompt, it used the Obsidian MCP to create a markdown file in Obsidian.

### gemma4:31b-cloud

**-** Weak reliability for code changes.

**-** Confidently hallucinates.

### qwen3.5:cloud

**-** Slower than the others.

**-** Somewhat verbose.

**-** Didn't really stand out compared to Kimi or DeepSeek.

## Vulnerability scan experiment

One task I wanted to test was a full application security review.

The process was simple:

1. Each LLM scans the entire codebase (frontend, backend, APIs, inputs, and database interactions).
2. Merge findings into a single report.
3. Give merged report to each model in turn and ask it to critically review the findings for false positives.

Models tested:

* Kimi
* DeepSeek
* MiniMax
* GLM

The output was large from all models, but especially MiniMax.

Below are a few examples of findings that turned out to be incorrect or overstated.

| Sample from source | Source  | Verdict    | Reason                                |
| --------------------------- | ------- | ---------- | ------------------------------------- |
| #1 uncaught throw           | Kimi    | INVALID    | Throw is caught by try/catch          |
| [H-2] user_id bypass        | MiniMax | INVALID    | Query param ignored; uses auth userId |
| [H-1] spam summarize-titles | MiniMax | OVERSTATED | Per-user rate limit still blocks      |
| [L-3] contact form URLs     | MiniMax | INVALID    | No user URLs in contact email body    |
| Stored XSS from RSS HTML    | Kimi    | OVERSTATED | Not rendered; future hypothetical     |
| javascript: in email        | MiniMax | OVERSTATED | Email clients block js: hrefs         |

**There were many more** examples than shown above.

The common pattern was that models often analyzed isolated code sections and made assumptions about the rest of the application. Several reports even confused local development environments with production environments.

## The good parts
To give them credit, they did discover real problems as well, including unsanitized inputs and logging sensitive data in plain text on the backend.

The useful findings only became obvious after heavy deduplication and multiple rounds of review. I ran roughly three review sessions with each model, asking them to criticize previous reports. That's twelve review passes in total.

The most valuable insight from the experiment was not the report itself, but the observation of how effective it is for models to verify each other's claims.

Despite the duplication, hallucinations, and false positives, the process still produced useful results.

For a future test I'd like to include ChatGPT as well, especially since I've seen claims that ChatGPT monthly subscriptions can be used to power Hermes. Remains to be seen if It's still so when I get to try it.

Have you done or seen similar comparisons? What were your results?