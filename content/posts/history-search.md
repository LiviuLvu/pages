---
title: "Searching my AI conversation history with Python, SQLite and MCP"
date: "2026-10-12"
summary: "One local archive for OpenCode and Hermes conversations. How Python gathers the history, SQLite indexes it, and an MCP service lets my agents retrieve earlier decisions and fixes."
tags: ["Home Lab", "Second brain", "Agents", "Python", "SQLite", "MCP"]
author: "Liviu Iancu"
series: ["AI"]
draft: true
---

Working with AI agents leaves a useful trail of references. The problem is finding any of it later. I can write important decisions into project notes, but not every debugging session deserves a manually maintained document.

Using multiple LLM harnesses in the last 3 years, complicates the problem of finding something months, or years back, that I know was at one point discussed and need to check details again.

Goal: get an answers based on my chat history from at least 2 of my most used harnesses: Hermes and Opencode.

I built **history-search**, a small local Python service that imports both sources into SQLite and exposes search through MCP. For now, no vector database and semantic search, only indexing.

## The layout

There are two flows. One gathers the history. The other retrieves it when I ask.

```text
OpenCode -> authenticated opencode api CLI --+
                                            |
Hermes   -> read-only SQLite snapshot -------+
                                            |
                                            v
                                      Python importers
                                            |
                                            v
                                      SQLite archive
                                      + FTS5 index

My recall request
  -> history-search skill instructions
  -> MCP search and conversation tools
  -> relevant archived messages
  -> LLM answer with source references
```

The parts:

| Part | Job |
| :--- | :--- |
| Python 3.11+ | Import, normalize, search and serve the history |
| SQLite | Store conversation metadata and retained text |
| SQLite FTS5 | Index words and rank matching records |
| MCP Python SDK / FastMCP | Expose four tools over localhost HTTP |
| Agent skill | Explain when to search and how to verify results |
| Command-line interface | Sync, search, inspect status, read and purge sessions |

## How the data is gathered

The archive makes its own copy. Runtime data and logs stay outside Git.

### OpenCode: use the API, not its internal database

The OpenCode importer calls the authenticated `opencode api` CLI. It walks both pagination levels: the session list, then the messages in each session.

Each imported session keeps its source ID, title and working directory. Each text field keeps its message ID, timestamp, role and field type. That gives a search result enough information to point back to a specific place in a conversation.

The importer compares the session's update timestamp with the value from the previous import. Unchanged sessions are skipped. A failed session is not marked as successfully imported, so it remains eligible for retry.

This avoids coupling the importer to OpenCode's internal database layout. It does depend on the CLI and API being available to the background service.

### Hermes: read a consistent SQLite snapshot

Hermes stores its current history in `~/.hermes/state.db`. The importer opens that database in read-only mode and starts a read transaction, giving it a consistent snapshot while it reads sessions and messages.

Hermes message edits do not have a reliable update timestamp for this import path. Instead of guessing which messages changed, the importer reconciles the retained text on every cycle. If a message is unchanged, it does not rewrite the archive or its search index.

Stored inactive or compacted messages are included when they still exist in the source. Binary attachments, backup files and unrelated development databases are not imported.

### More than the final answer

Both importers retain the available text fields, including:

- user and assistant messages;
- stored reasoning text;
- tool inputs, including commands;
- tool results and errors;
- source truncation flags where available.

Tool results matter. An assistant saying “the fix works” is not the same as a recorded passing test or my confirmation. Keeping the surrounding evidence makes the archive more useful than a collection of final answers.

There is still a hard limit: if a source already truncated a tool result, this service cannot recover the missing text.

## How SQLite indexes the history

The archive separates sessions from message fields. A single message can contain several fields: ordinary text, reasoning, a tool input and a tool result. These become separate searchable entries, with their original order retained.

SQLite's **FTS5** extension builds an inverted index over that text. Think of it as a lookup from a word to the entries containing it. Search does not need to scan every conversation from beginning to end.

Insert and delete triggers keep the index aligned with the archive. When an imported message changes, its old entries are replaced. Unchanged messages are left alone.

Results are ranked with **BM25**, a keyword relevance score that considers term frequency, term rarity and text length.

For example, the CLI can search a phrase or alternative words:

```bash
.venv/bin/history-search search '"docker compose"'
.venv/bin/history-search search 'invoice OR receipt' --source hermes
```

FTS5 supports words, quoted phrases, boolean expressions and prefixes. It does not provide arbitrary substring matching. If I use completely different words from the original conversation, the agent may need to try synonyms.

Filters can narrow results by source, session directory, role, field type or date. The directory filter means where the conversation ran, not what project it discussed. A conversation started in one folder can still contain useful information about another project.

For this use case, keyword search is a reasonable starting point. Error messages, command names and project names are good search terms. Semantic search could help with vague recollections, but it would add an embedding model and another indexing path to maintain.

## MCP: how the agent reaches the archive

MCP, or Model Context Protocol, lets an AI client discover and call external tools. Here it connects OpenCode and Hermes to the same local service:

```text
http://127.0.0.1:8767/mcp
```

The service uses Streamable HTTP and exposes four tools:

| Tool | What it returns |
| :--- | :--- |
| `search_history` | Ranked snippets, source references and archive status |
| `get_conversation` | A bounded page of message fields, optionally centered on a search hit |
| `history_status` | Counts, last successful imports, failures and retained truncation flags |
| `refresh_history` | An explicit import of both sources |

Search snippets help find a candidate. They are not enough to establish what happened. The agent reads messages around the match and follows later pages when it needs to verify the outcome.

Conversation retrieval is bounded so one large session does not fill the model's context window. Long text fields are returned in portions of up to 8,000 characters, with offsets for reading the rest. Response clipping is marked separately from truncation that already existed in the source.

MCP calls run in worker threads. An explicit refresh (of the history index) does not block the server's event loop and make every search wait for the import to finish. Searches use the archive as it currently stands.

The endpoint binds only to loopback (the host machine). I do not expose my conversation archive on the LAN.

## The skill

The history-search skill is a small Markdown file installed in harness skill directory. It does not import data or run the search engine. It tells the agent how to use the MCP tools.

The main rule: **search history only when I explicitly ask for recall**. An ordinary coding task should not silently pull old conversations into its context.

For a request such as “Find how we fixed that Docker issue before,” the intended sequence is:

1. Search all sources unless I request a narrower scope.
2. Read the conversation around a matching message.
3. Check subsequent messages for tests, corrections or confirmation.
4. Check archive freshness and import gaps.
5. Answer with the source, conversation title, session ID and message ID.

Historical messages are evidence, not current instructions. An old tool result containing a command does not authorize running it now. An old assistant claim does not prove a fix succeeded.

This distinction is easy to overlook. Without it, search could turn yesterday's confident mistake into today's accepted fact.

## Where the LLM fits

There is no LLM inside history-search. Python gathers the text. SQLite finds matches. MCP carries the request and response.

The LLM inside the harness chooses search terms, reads the returned context and writes the answer. Only the retrieved portions enter the current conversation; the whole archive is not loaded into every prompt. Nothing here retrains the model or gives it permanent memory.

## Keeping the host service running

The Python service imports at startup, then waits sixty seconds after each completed import before starting the next cycle. A large import can therefore make freshness exceed one minute. The status tool reports the actual last successful sync instead of pretending this is real-time replication.

A thread lock and a file lock prevent competing imports, including one started from the CLI while the daemon is already syncing. An overlapping refresh returns `already_running`.

On macOS, launchd manages the service. I considered using a container but for this first version I just wanted to try out the oncept. 

Containerizing the MCP is on my todo list until the service proves stable and genuinely useful for me. Main advantage I see for containerization is host reboot management and verification is much simpler and independent of host OS.

## A failed import that looked worse than it was

One OpenCode message page repeatedly failed JSON decoding. It would have been easy to treat this as a damaged conversation or source database.

The actual problem was the CLI response capture. A large response was truncated when stdout was captured through a pipe. The same API request produced valid JSON when its output was captured through a temporary file.

The importer now tries the normal pipe first. If UTF-8 or JSON decoding fails, it retries the same request through an automatically removed temporary file. If decoding still fails, pagination retries with smaller pages. A page that remains unreadable is reported as an import gap, not treated as an empty conversation.

After the change and daemon restart, the documented verification found zero import failures for both sources. The previously failing message's seven fields matched the API and were searchable and retrievable.

That is a recorded validation result, not a promise that future imports cannot fail. Current status remains the place to check coverage.

The project also has automated tests for import and retrieval behavior, plus an MCP smoke test that searches both sources and retrieves matching context. Client checks confirmed that both OpenCode and Hermes could use the tools.

## What the archive keeps, and what it does not

Deleting a conversation in OpenCode or Hermes does not delete the copy already imported here. This is intentional: source cleanup should not silently erase an earlier solution from the archive.

Edits are different. Imported edits replace the current archived text. There is no revision history.

The CLI supports explicit per-session purge, but a session still present at the source will be imported again on the next sync. Purging the archive does not change either source.

This is not a complete backup system. It does not import backup stores, recover missing attachments or reconstruct truncated output. A successful import says it read the available source records; it does not prove that no older history was lost before this project existed.

## Conclusion

The useful part is not giving an agent more “memory” in the abstract. It is giving it a way to find a specific earlier conversation, inspect what actually happened and show me where the answer came from.

For now, Python, SQLite and four MCP tools are enough. The skill keeps recall deliberate, and launchd keeps the service running.

I still write important decisions into project notes. This archive fills the gap between those maintained notes and the much larger trail of everyday conversations.

Hope this gives you a useful starting point for building your own agent history search. Have fun building something new!
