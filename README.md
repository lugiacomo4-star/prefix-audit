# prefix-audit

I asked Claude Code to reply with the single word `ok`.

It cost **56,105 input tokens**. Two of them were my question.

```
  API calls              1
  input tokens (total)   56,105
  smallest single call   56,105   <- roughly your fixed prefix

  per call:
     1  input        2   cache created   43,160   cache read   12,943   =    56,105
```

That 56,105 is the prefix: the system prompt plus the full description of every
tool you have connected, re-sent on every single API call, whether the model
touches those tools or not. On a multi-step task you pay it once per call.

> **Correction, 16 November 2026 — Claude Code 2.1.325.** The "full description
> of every tool" half of that sentence is no longer true on this build. Tool
> *schemas* are not in the prompt any more; only tool *names* are, and the
> schemas load on demand. The prefix is still re-sent on every call — it just
> carries a list instead of a catalogue. Measured below with `mcp-name-tax`.

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

- One task, one repo. The `ok` floor is stable to within ~1% across runs (56,105
  and 56,579 on two runs; the difference is cache warmth). The comparison table
  below is a single afternoon and I no longer have its raw stream-json.
- Dollar cost is not a clean comparison — it depends how warm your cache already
  was. Input tokens are.
- My task was short. On a long session that reads many files, the 1.9%
  content-compression share should grow. I don't know where it crosses over.
- `claude` must be able to run non-interactively. Building the code graph from
  *inside* Claude Code failed outright: every shell command stopped for an
  approval nobody was there to give, 17 wasted API calls.

## Plugin cost: the per-plugin number is honest, the sum isn't

`claude plugin details <name>` prints an "always-on" token figure: what that
plugin adds to every session. I checked it against the real API bill with the
`ok` prompt on this machine, Claude Code 2.1.294, nothing switched on but the
plugin under test. Each configuration was measured twice and returned the same
total to the token.

| configuration | input tokens | delta | `plugin details` claims |
|---|---|---|---|
| no plugins | 41,458 | — | — |
| plugin A alone (1 skill) | 41,756 | +298 | ~303 |
| plugin B alone (2 skills) | 41,714 | +256 | ~259 |
| A and B together | 41,694 | **+236** | ~562 |

Per plugin the estimate is good: 303 vs 298, 259 vs 256, both within 2%.

Together they bill 236 — less than A alone, less than B alone. Adding a plugin
made the prefix smaller. I can't explain it. The deltas are reproducible, so it
isn't cache warmth, and the enabled/disabled state was verified with
`claude plugin list` before each measurement.

The practical consequence: that column can't be added up. "What are my plugins
costing me" is only answerable by measuring the whole prefix with your actual
set switched on.

Separately, from the same run: the scaffold `claude plugin init` generates
reports `Skills (0)` and `~0 tok`. It costs **71 tokens** of prefix on every
session — its root `SKILL.md` is loaded but not counted as a component.

### Repeat it

```sh
./prefix-audit "Reply with exactly: ok"   # your floor, as configured now
claude plugin disable <plugin>            # one at a time
./prefix-audit "Reply with exactly: ok"   # same prompt, measure again
claude plugin list                        # confirm the state you think you're in
```

If the sum breaks on your machine too, I'd like the three numbers: A, B, A+B.

## The catalogue left the prompt. The names stayed.

`prefix-audit` measures your prefix. It doesn't tell you what's *in* it. So I
built a throwaway MCP server that answers `tools/list` and nothing else, pointed
Claude Code at it with `--mcp-config`, and changed one variable at a time.

Claude Code 2.1.325, Opus 5.5, 40 stub tools, same trivial prompt each time.
Each row is the whole prompt: `input` + `cache_creation` + `cache_read`.

| 40 stub tools | schema bytes each | prompt tokens | delta | per tool |
|---|---|---|---|---|
| none connected (baseline) | — | 44,887 | — | — |
| names `t0`…`t39` | 525 | 45,371 | +484 | 12.1 |
| names `get_record_0`… | 535 | 45,485 | +598 | 14.9 |
| names `retrieve_workspace_record_by_identifier_with_history_variant_0`… | 585 | 46,289 | +1,402 | 35.0 |
| names `get_record_0`…, fatter schema | 2,853 | 45,488 | +601 | 15.0 |

Two readings:

**Schema size costs nothing.** Row 3 and row 5 are the same 40 tools with the
same names. Row 5's schemas are 5.3x larger — 2,853 bytes against 535, about 93 KB of extra JSON
across the server. It bought 3 extra tokens of prompt, 0.5%.
The schemas are not being sent. They load when the model goes looking for them.

**Name length costs plenty.** Rows 2 to 4 keep the description and the parameters
identical and change only the names. Two characters to 62 characters moves the same 40 tools from
484 tokens to 1,402. Per tool, 12 to 35.

What's in your prompt is a list of names. That is also all the model has when it
decides whether your tool is worth opening.

Stable across runs: repeated measurements landed within ~5 tokens of each other.

### Repeat it

```sh
./mcp-name-tax             # the table above, on your build
./mcp-name-tax --tools 80  # heavier sweep
./mcp-name-tax --json      # machine-readable
```

It needs `claude` and `python3`, takes about a minute, and talks to no network
service that Claude Code wasn't already talking to. The stub server exposes no
capability: it answers `tools/list` and ignores everything else.

If your build still ships the full schemas, the script says so in plain words —
and I'd like to hear it, because then this is environment-specific and not a
version change.

## 100 skills cost less than 10

The advice everywhere is to keep your skills folder small so it doesn't eat your
context. `skill-tax` measures it. It builds a throwaway project with N skills in
`.claude/skills/` - real frontmatter description, 200-line body each - asks the
same trivial question, and reads the tokens off the stream-json. Then it runs the
same project with `--disable-slash-commands`, which switches the whole skill and
command catalogue off. The gap between those two numbers is what the catalogue
costs you on every API call.

Claude Code 2.1.328, Opus 5.5, Linux container, empty project, no CLAUDE.md.

```
prompt: 'say ok'   (prompt tokens = input + cache_creation + cache_read)

  skills   catalogue          disk  prompt tokens  catalogue
  100      off             2491.9K         43,702          -
  0        on                 0.0K         45,488     +1,786
  10       on               249.2K         45,495     +1,793
  100      on              2491.9K         44,870     +1,168
```

Ten skills cost 7 tokens. A hundred skills - 2.4 MB of `SKILL.md` on disk - cost
618 tokens *less* than none.

Two readings:

**The bodies are not in the prompt.** 2.4 MB of skill text on disk, and the
prompt went down. Only the catalogue line is ever sent; the body loads when the
skill is invoked.

**The catalogue does not grow with your folder.** It sat between 1,168 and 1,793
tokens whether the project had 0 skills or 100. Note that the 0-skill row is not
zero: that machine already had the built-in commands and a set of plugin skills.
Read the deltas between rows, not the absolute numbers.

So the price of a big skills folder is not tokens. What I can't tell yet is what
gets dropped when the allowance is full - the oldest skill, the longest
description, or something else. The `--disable-slash-commands` floor was the
stablest number in the whole repo: measured by hand twice, once with 10 skills in
the project and once with 100, it came back as 43,770 both times, to the token.
The script's own run landed at 43,702.

### Repeat it

```sh
./skill-tax                     # the table above, on your build
./skill-tax --skills 0,50,200   # pick your own counts
./skill-tax --json              # machine-readable
```

Each count is one API call, plus one for the floor. The generated skills live in
a temp directory and are deleted on exit.

## What I'd like

Run the `ok` prompt and tell me your floor, with a rough count of how many MCP
servers you have switched on. I want to know whether ~56k is normal or whether
I just have too many connected. On a second machine with fewer servers connected
the same prompt cost 38,837, so the spread is real and I'd like to map it.

Run `./mcp-name-tax` too and tell me your tokens per tool, with a rough count of
how many tools you have switched on. I measured 15 in a Linux container; I don't
know whether that number travels.

MIT.
