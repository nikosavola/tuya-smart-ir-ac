## Fix: integration fails to load (`NameError: Callable`)

`bridge.py` annotated a parameter with `Callable` but never imported it, and the
module doesn't use `from __future__ import annotations` — so Python evaluated the
annotation at import time and threw `NameError`. Because `__init__.py` imports the
bridge unconditionally, this took down the **whole** integration, not just Pulsar.

One-line fix: import `Callable`/`Awaitable` from `collections.abc` (and tightened
the handler type while I was there).

**Risk:** tiny. No behavior change beyond "it loads now."
