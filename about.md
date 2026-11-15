---
layout: page
title: About
permalink: /about/
---

Backend engineer. Python, mostly. I care about the parts of a system that fail
quietly: message delivery, caches, retries, the second request that does not
look like the first.

Things I build:

- [txoutbox](https://github.com/OsmnvAslan/txoutbox) — transactional outbox relay for asyncio, no dependencies.
- [promptcachelint](https://github.com/OsmnvAslan/promptcachelint) — explains why an LLM prompt cache missed.

Upstream:

- CPython: [fix for `asyncio.as_completed()` outside a running event loop](https://github.com/python/cpython/pull/157857), [dead code removal in `asyncio.windows_events`](https://github.com/python/cpython/pull/160241).
- Django: [clearer `path()` / `re_path()` error](https://github.com/django/django/pull/21820), [security policy page](https://github.com/django/django/pull/22052), [`JsonResponse` docs](https://github.com/django/django/pull/21725).

GitHub: [OsmnvAslan](https://github.com/OsmnvAslan) · PyPI: [txoutbox](https://pypi.org/project/txoutbox/), [promptcachelint](https://pypi.org/project/promptcachelint/) · Mail: osmnvaslan@gmail.com
