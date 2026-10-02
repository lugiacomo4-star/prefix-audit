# prefix-audit

I asked Claude Code to reply with the single word `ok`.

It cost **38,837 input tokens**. Two of them were my question.

```
  API calls              1
  input tokens (total)   38,837
  smallest single call   38,837   <- roughly your fixed prefix

  per call:
     1  input        2   cache created   38,835   cache read        0   =    38,837
```

That 38,835 is the prefix: the system prompt plus the full description of every
tool you have connected, re-sent on every single API call, whether the model
touches those tools or not. On a multi-step task you pay it once per call.

This repo is one script that tells you your own number.

## Usage

```sh
./prefix-audit "Reply with exactly: ok"          # measure the floor
./prefix-audit "explain how routing works here"  # measure a real question
./prefix-audit "your question" --keep            # keep the raw stream-json
```

No dependencies beyond `claude`, `bash` and `python3`. It runs
`claude -p --output-format stream-json`, then sums `input_tokens` +
`cache_creation_input_tokens` + `cache_read_input_tokens` across every API call
in the round — which is the number you actually pay for, and not the one the
cost line draws your eye to.

Run the `ok` prompt first. Whatever it returns is your floor: the price of asking
Claude Code anything at all, in that directory, with that set of MCP servers
switched on. Then run a real question and compare.

## Why the floor is the interesting number

Measured on a 27-file / 311 KB Node app, one question about one route, warm cache,
Claude Code 2.1.283 with Opus 5.5:

| setup | API calls | tools used | input tokens |
|---|---|---|---|
| Claude Code alone | 6 | 4 (grep, Read) | 243,459 |
| + a prefix-trimming proxy | 6 | 4 (grep, Read) | **86,337** |
| + a code-graph tool, asked normally | 6 | 4 (Grep, Read) | 260,853 |
| + the same tool, called explicitly | 9 | 5 | 542,738 |

All four answers were correct and nearly identical — same file, same line.

A 64.5% cut, same task, same answer. I assumed the proxy was compressing tool
output: truncating grep results, summarising files. It barely was. Over 10
requests its own stats attributed **8,280 tokens (1.9%)** to content compression
and **364,952** to dropping tool schemas from the prompt. Per-call prefix went
from ~40,600 tokens to ~14,400.

So the saving isn't compression. It's subtraction — and the thing being
subtracted is a catalogue, not your code.

The code graph didn't help here, which is worth saying because it sounds like it
should. Building it is genuinely cheap: 321 nodes, 578 edges, 2 seconds, zero
tokens, all local. But for a question about one route its query returned 65 nodes
(~5,800 tokens) and Claude then grep'd and read the two files anyway. It stacked
on top of the reading instead of replacing it, and its skill file added ~3,000
tokens of system prompt per call. I'd expect it to earn its keep on a large
codebase and an architecture question, not on 27 files.

## Caveats, because they decide how much this is worth

- One task, one repo. The `ok` floor reproduces exactly (38,837 twice); the
  comparison table is a single afternoon.
- Dollar cost is not a clean comparison — it depends how warm your cache already
  was. Input tokens are.
- My task was short. On a long session that reads many files, the 1.9%
  content-compression share should grow. I don't know where it crosses over.
- `claude` must be able to run non-interactively. Building the code graph from
  *inside* Claude Code failed outright: every shell command stopped for an
  approval nobody was there to give, 17 wasted API calls.

## What I'd like

Run the `ok` prompt and tell me your floor, with a rough count of how many MCP
servers you have switched on. I want to know whether ~38k is normal or whether
I just have too many connected.

MIT.
