# Route your first requests

In this tutorial you build a Router with three mock deployments and watch it balance load, retry, cool a deployment down and fall back to another model group. Every deployment uses `mock_response`, so nothing leaves your machine and you need no API keys

You need a checkout of this repo with its environment installed (`uv sync`). Run each step from the repo root with `uv run python`, or paste the steps one after another into a single file

```mermaid
flowchart LR
    s1["1. Build a Router"] --> s2["2. Send requests"] --> s3["3. Look inside"] --> s4["4. Break a deployment"] --> s5["5. Break the group"]
```

## 1. Build a Router

Create `tutorial.py` with a model list. Two deployments, `a` and `b`, serve the model group `chat`. A third, `c`, serves the group `backup`. The router falls back from `chat` to `backup` and retries once inside a group before it does

```python
import asyncio
from collections import Counter

from litellm import Router
from litellm.router_api import ChangeSet
from litellm.types.router import Deployment


def mock(id: str, group: str, response: str) -> dict:
    return {
        "model_name": group,
        "litellm_params": {"model": "openai/gpt-4o-mini", "api_key": "sk-fake", "mock_response": response},
        "model_info": {"id": id},
    }


router = Router(
    model_list=[
        mock("a", "chat", "hi from a"),
        mock("b", "chat", "hi from b"),
        mock("c", "backup", "hi from backup"),
    ],
    fallbacks=[{"chat": ["backup"]}],
    num_retries=1,
)


async def ask() -> tuple[str, str]:
    response = await router.acompletion(model="chat", messages=[{"role": "user", "content": "hello"}])
    return response._hidden_params["model_id"], response.choices[0].message.content
```

`model_name` is the name callers send. `litellm_params` is what the router passes to `litellm.acompletion` once it picks the deployment. `model_info.id` names the deployment, and the router hashes the group and params into an id when you leave it out

## 2. Send requests

Add a `main` coroutine and send twenty requests

```python
async def main() -> None:
    print("picked:", Counter([(await ask())[0] for _ in range(20)]))


asyncio.run(main())
```

Run it. You see both deployments picked, in a split close to even

```
picked: Counter({'b': 12, 'a': 8})
```

No strategy was configured, so the router used `simple-shuffle`. Neither deployment sets `weight`, `rpm` or `tpm`, so each pick is a uniform random choice between the healthy deployments of the group

## 3. Look inside

The router answers questions about itself through its entry points. Add these lines to `main`

```python
    registry = router.registry()
    print("groups:", sorted(registry.names(kind="group_set")))
    print("chat deployments:", [d["model_info"]["id"] for d in registry.deployments("chat")])
    print("strategy:", router.settings().routing_strategy)
    picked = await router.select("chat")
    print("select picked:", picked["model_info"]["id"])
```

```
groups: ['backup', 'chat']
chat deployments: ['a', 'b']
strategy: simple-shuffle
select picked: a
```

`registry()` holds the configured deployments, `settings()` is a frozen snapshot of the configuration, and `select()` runs the same selection a real request runs without calling the provider. [Entry points](../reference/entry-points.md) lists everything they offer

## 4. Break a deployment

Replace deployment `a` with one that always answers 429. Changes go through `apply()` with a `ChangeSet`, which bumps the registry version. The exact number depends on how many deployments were added before, what matters is that it goes up

```python
    broken_a = Deployment(**mock("a", "chat", "litellm.RateLimitError"))
    print("version after apply:", router.apply(ChangeSet(upsert=[broken_a])).version)
    print("picked:", Counter([(await ask())[0] for _ in range(10)]))
    print("cooldowns:", await router.health().cooldowns.async_get_active_cooldowns(
        model_ids=["a", "b", "c"], parent_otel_span=None
    ))
```

```
version after apply: 8
picked: Counter({'b': 10})
cooldowns: [('a', {'exception_received': 'litellm.RateLimitError: ...', 'status_code': '429', 'timestamp': ..., 'cooldown_time': 5})]
```

Every request succeeded. The first one that landed on `a` failed with a 429, and two things happened

```mermaid
sequenceDiagram
    participant R as Router
    participant A as deployment a
    participant B as deployment b
    participant S as DualCache
    R->>A: attempt 1
    A-->>R: 429 RateLimitError
    R->>S: failure callback sets deployment:a:cooldown (TTL 5s)
    R->>R: retry 1 of 1, skip a
    R->>B: attempt 2
    B-->>R: 200
    Note over R,S: later requests read the cooldown and only see b
```

The router retried in the same group and skipped `a`, so the caller got `b`'s answer. The failure callback also put `a` into cooldown, because a 429 in a group with more than one deployment cools that deployment at once. For the next five seconds, the default cooldown time, selection filters `a` out before the strategy ever sees it. [Retries, fallbacks and cooldowns](../explanation/reliability.md) explains the rules

## 5. Break the group

Break `b` too and send one more request

```python
    broken_b = Deployment(**mock("b", "chat", "litellm.RateLimitError"))
    router.apply(ChangeSet(upsert=[broken_b]))
    print("answer:", await ask())
```

```
answer: ('c', 'hi from backup')
```

With both `chat` deployments failing, the retry has nowhere left to go in the group. Once retries are spent, the fallback wrapper looks up `chat` in `fallbacks`, finds `backup`, and runs the whole retry loop again against that group

```mermaid
flowchart TB
    req["acompletion(model='chat')"] --> fb["Fallback wrapper"]
    fb --> rt1["Retry loop in 'chat'"]
    rt1 -->|"a: 429, b: 429"| fb
    fb -->|"fallbacks: chat to backup"| rt2["Retry loop in 'backup'"]
    rt2 -->|"c: 200"| done["'hi from backup'"]
```

## What you saw

A Router owns a list of deployments grouped by `model_name`. Each request is resolved to a group, the group's deployments are filtered, and a strategy picks one. Failures feed back into shared state (the cooldown you saw) and into the retry and fallback wrappers around the call. Code outside the router reads and changes all of this through a handful of entry points

Next, read [The life of a request](../explanation/request-lifecycle.md) for the full call stack, or [Configuration and defaults](../reference/configuration.md) for the knobs you skipped here
