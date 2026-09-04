# Condition-Based Waiting

Replace arbitrary sleeps with condition polling. A sleep is a guess about how long something takes — a condition check is a fact.

## The Rule

**Never use `sleep()` for synchronization. Use condition polling with explicit timeout and failure on breach.**

A sleep that is "long enough" today becomes a flaky test in CI tomorrow. A condition poll succeeds as soon as the condition is true and fails explicitly at the timeout boundary.

## Polyglot Patterns

**Bash:**
```bash
deadline=$((SECONDS + 10))
while ! <condition_command>; do
  [ $SECONDS -ge $deadline ] && { echo "ERROR: timeout waiting for <condition>"; exit 1; }
  sleep 0.1
done
```

**Python (sync):**
```python
import time

def wait_for(condition_fn, timeout=10, interval=0.1, msg="condition"):
    deadline = time.monotonic() + timeout
    while not condition_fn():
        if time.monotonic() >= deadline:
            raise TimeoutError(f"Timed out waiting for {msg}")
        time.sleep(interval)
```

**Python (async):**
```python
import asyncio

async def wait_for_condition(condition_coro, timeout=10):
    await asyncio.wait_for(condition_coro(), timeout=timeout)
```

**Node.js:**
```javascript
function waitFor(conditionFn, timeoutMs = 10000, intervalMs = 100) {
  return new Promise((resolve, reject) => {
    const deadline = Date.now() + timeoutMs;
    const check = () => {
      if (conditionFn()) return resolve();
      if (Date.now() >= deadline) return reject(new Error(`Timeout waiting for condition`));
      setTimeout(check, intervalMs);
    };
    check();
  });
}
```

**Rust:**
```rust
use std::sync::{Mutex, Condvar};
use std::time::Duration;

let (lock, cvar) = &*pair;
let mut state = lock.lock().unwrap();
let result = cvar.wait_timeout_while(state, Duration::from_secs(10), |s| !s.ready).unwrap();
if result.1.timed_out() {
    panic!("Timed out waiting for condition");
}
```

## Timeout Budget

| Test type | Max wait | Failure behavior |
|-----------|---------|-----------------|
| Unit test | 10s | Explicit error, not a pass |
| Integration test | 60s | Explicit error, not a pass |
| E2E test | 300s | Explicit error, not a pass |

Exceeding the budget means the test infrastructure is broken, not that the test should wait longer.

## Common Violations to Fix

| Violation | Replace with |
|-----------|-------------|
| `sleep(5)` after spawning a process | Poll for process readiness (PID exists, port open, health endpoint responds) |
| `sleep(1)` in test setup | Wait for the specific resource the setup creates |
| `time.sleep(0.5)` between retries | Condition poll with explicit timeout |
| `Thread.sleep(200)` to "let async settle" | Await the actual async operation or poll for its observable effect |
| `await new Promise(r => setTimeout(r, 100))` | Wait for the DOM change, API response, or state mutation directly |

## The Test for Your Wait

Ask: if the condition resolves in 1ms, does your wait code handle it? If the condition never resolves, does your wait code fail explicitly with an error? If the answer to either is no, the wait is wrong.
