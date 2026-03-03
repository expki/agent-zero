## Summary
This PR fixes a memory-recall timeout failure mode that could interrupt the agent loop.

## Context
The contribution guide asks for clear purpose, scope, and testing details. This PR targets `development` and focuses on recall stability.

Note on file path naming: in the current plugin layout, the timeouted recall work is implemented in `plugins/memory/extensions/python/message_loop_prompts_after/_50_recall_memories.py` (not `python/extensions/process_chain_end/_50_process_queue.py`).

## Problem
Memory recall is wrapped with `asyncio.wait_for(...)`. When embedding/vector search is slow, the task can time out and raise `TimeoutError` via `CancelledError`. If that exception propagates, it can fail prompt preparation and disrupt normal monologue flow.

## Changes
- Increased recall timeout window:
  - `plugins/memory/extensions/python/message_loop_prompts_after/_50_recall_memories.py`
  - `SEARCH_TIMEOUT: 30 -> 120`
- Made recall wait non-fatal and observable:
  - `plugins/memory/extensions/python/message_loop_prompts_after/_91_recall_wait.py`
  - Wrapped `await task` in `try/except (TimeoutError, CancelledError)`
  - Logs warning with formatted error + elapsed wait time in `RecallWait.execute`

## Why increase `SEARCH_TIMEOUT`
The timeout budget covers more than a single FAISS call. It includes:
- embedding generation for the query
- similarity search calls across memory areas
- optional model-assisted query prep / filtering depending on config

In real deployments (provider latency, transient network slowness, host contention), 30s is frequently too tight for this full path. Raising to 120s reduces false timeouts while preserving an upper bound so recall cannot run indefinitely.

## Why memory recall should be best-effort
Memory recall is an enhancement, not a hard prerequisite for producing a response.

If recall failure is treated as fatal, temporary slowness in a non-critical subsystem can block or fail the main interaction loop. That couples availability of core chat behavior to optional recall quality.

Making recall best-effort preserves core reliability:
- agent still responds even if recall times out
- timeout is still visible via warning logs
- operators retain diagnostics (including elapsed wait time) without taking down user-visible behavior

## Testing
- Verified Python syntax for changed files with `py_compile`.
- Manual code-path validation confirms:
  - timeout still enforced in recall task
  - timeout/cancel no longer propagates out of `RecallWait.execute`
  - warning log includes elapsed execution time

## Risk / Compatibility
- Low risk and backward-compatible.
- Behavior change is intentionally narrow: only timeout handling in recall wait is changed from fail-fast to degrade-gracefully.
