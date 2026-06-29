## Stability: keep the Pulsar reconnect task alive + tidy dispatch

The reconnect loop was a bare `asyncio.create_task(...)` with the handle thrown
away, so the GC could quietly kill it and silently stop real-time updates. Now we
hold the reference and cancel it on stop.

Also: dispatch Pulsar messages via `async_create_task` instead of legacy `add_job`,
and allow multiple handlers per device id. (Includes the `Callable` import so the
file loads on its own — overlaps harmlessly with the bridge-callable-import branch.)
