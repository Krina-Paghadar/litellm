# Change deployments or settings at runtime

Use this guide to add, replace or remove deployments, or to change router settings, on a Router that is already serving traffic. All changes go through `router.apply()` with a `ChangeSet`. Do not assign Router attributes or mutate the lists that `settings()` and `registry()` return

```mermaid
sequenceDiagram
    participant C as Your code
    participant R as Router
    participant Reg as Registry
    C->>R: apply(ChangeSet(...))
    R->>R: assign, then settings, then upsert, then delete, then redis_cache
    R->>Reg: version += 1 per deployment change
    R-->>C: ApplyResult(version, upserted, deleted)
    Note over R: the cached settings() snapshot is dropped,<br/>the next read builds a new one
```

## Add or replace a deployment

`upsert` matches on `model_info.id`. A deployment with a new id is added, one with an existing id replaces the old one

```python
from litellm.router_api import ChangeSet
from litellm.types.router import Deployment

result = router.apply(ChangeSet(upsert=[
    Deployment(
        model_name="gpt-4o",
        litellm_params={"model": "azure/gpt-4o-westus", "api_base": "...", "api_key": "..."},
        model_info={"id": "gpt-4o-westus"},
    )
]))
previous = result.upserted[0]  # the deployment it replaced, or None
```

## Remove a deployment

```python
result = router.apply(ChangeSet(delete=["gpt-4o-westus"]))
removed = result.deleted[0]  # the removed deployment, or None if the id was unknown
```

## Change settings

Use `settings` for the validated runtime settings that `update_settings` accepted, and `assign` to replace a field from `SETTING_FIELDS` as it is

```python
router.apply(ChangeSet(settings={"num_retries": 3, "routing_strategy": "latency-based-routing"}))
router.apply(ChangeSet(assign={"cooldown_time": 30, "fallbacks": [{"gpt-4o": ["claude-sonnet"]}]}))
```

`assign` rejects names outside `SETTING_FIELDS` with a `ValueError`. `settings` is quieter, it skips keys outside `RUNTIME_UPDATABLE_ROUTER_SETTINGS` with only a debug log, so check the name if a change seems to do nothing. `settings` also casts the numeric ones, rebuilds routing groups when the strategy changes, and switches the strategy's loggers, which `assign` does not

## Changes that need I/O

Re-reading discovered model info is async, so it needs `aapply`

```python
await router.aapply(ChangeSet(refresh_model_info=True))
```

## Several changes at once

One `ChangeSet` can carry all of these. They are applied in the order `assign`, `settings`, `upsert`, `delete`, `redis_cache`, and `ApplyResult.version` is the registry version after the last one

## Old code you may find

| Old | New |
| --- | --- |
| `router.upsert_deployment(d)` | `router.apply(ChangeSet(upsert=[d]))` |
| `router.delete_deployment(id=i)` | `router.apply(ChangeSet(delete=[i]))` |
| `router.update_settings(**s)` | `router.apply(ChangeSet(settings=s))` |
| `router.cooldown_time = 30` | `router.apply(ChangeSet(assign={"cooldown_time": 30}))` |
| `router.update_redis_cache(cache=c)` | `router.apply(ChangeSet(redis_cache=c))` |
| `await router.arefresh_model_info()` | `await router.aapply(ChangeSet(refresh_model_info=True))` |

The old forms still work through `DeprecatedRouterMembers` and warn. Router settings are meant to be fixed after construction except through `apply`, so new code should not rely on attribute writes taking effect
