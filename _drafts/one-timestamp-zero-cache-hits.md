---
layout: post
title: "One timestamp, zero cache hits"
description: A prompt cache is a prefix match. One changing line in the system prompt and every request is billed in full.
---

<!--
DRAFT. Not published: Jekyll ignores _drafts/ unless run with --drafts.
Before moving this to _posts/:
1. Run `uv run --with anthropic examples/before_after.py` in promptcachelint (needs
   ANTHROPIC_API_KEY, costs a few cents) and paste the real numbers where marked TODO.
2. Add the date to the file name: YYYY-MM-DD-one-timestamp-zero-cache-hits.md
-->

Prompt caching on Anthropic and OpenAI is a prefix match. The provider hashes the request
from the start up to a cache marker, and a later request reads the cache only if that prefix
is identical. Change one character before the marker and everything after it is billed
again at full price.

The provider tells you that it happened: `cache_read_input_tokens: 0`. It does not tell you
where.

## The most common way to break it

```python
system = RULES + f"\nCurrent time: {now}"
```

The line looks harmless and is often added on day one. It sits at the end of a long system
prompt, inside the cached prefix, and it changes every request.

[promptcachelint](https://github.com/OsmnvAslan/promptcachelint) records requests, diffs
each one against the previous request of the same conversation, and names the block, the
offset and the characters that differ. On a conversation like the one above it prints:

```
session 600a924e2101 [anthropic] 3 req, hit ratio 0.0%, 2 break(s)
  #0  claude-opus-5  segments=2 cacheable=2  in=24 read=0 write=1601
      ! CL012 iso-datetime inside the cacheable prefix at system[0]
        → Timestamps change every request. Move them after the last breakpoint or drop them.
  #1  claude-opus-5  segments=4 cacheable=4  in=25 read=0 write=1602
      ✗ prefix broken at system[0] +6190 (~1553 tokens lost, estimate)
        was: '…ime: 2026-09-20T10:15'
        now: '…ime: 2026-09-20T10:16'
```

Every request writes about 1600 tokens to the cache and reads none of them back.

## Before and after

The repository has a script that runs the same six-turn conversation twice against the real
API: once with the time in the system prompt, once with that line moved into the latest
user turn.

TODO: paste the two reports and the billed difference from `examples/before_after.py`.

## What is measured and what is estimated

Token counts for reads and writes come from the provider's own `usage` fields and are exact.
"Tokens lost" is an estimate at about four characters per token, and the report labels it
as one.

## Try it

```bash
pip install "promptcachelint[httpx2]"
```

Wrap the SDK's transport, run your agent as usual, and print the report:

```python
recorder = promptcachelint.Recorder(promptcachelint.JsonlSink("trace.jsonl"))
claude = anthropic.Anthropic(
    http_client=anthropic.DefaultHttpxClient(
        transport=wrap_transport(httpx2.HTTPTransport(), recorder)
    )
)
# ... run your agent ...
print(promptcachelint.analyze(recorder.records).to_text())
```
