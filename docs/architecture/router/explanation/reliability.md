# Retries, fallbacks and cooldowns

The router has three tools for failures, and they work at different scopes. A retry tries the same model group again, and selection can land on a different deployment. A fallback moves the request to a different model group. A cooldown takes one deployment out of selection for every request, for a few seconds

```mermaid
flowchart TB
    subgraph fallbacks["Fallback wrapper: across model groups"]
        subgraph retries["Retry wrapper: inside one model group"]
            attempt["Attempt: select + call"]
        end
    end
    attempt -->|"failure"| cb["Failure callback"]
    cb -->|"maybe"| cool[("deployment:id:cooldown")]
    cool -.->|"filters every later selection"| attempt
```

## Retries

`async_function_with_retries` runs the first attempt. When it fails, the router works out how many retries this request gets, decides whether this error is worth retrying, waits, and loops

```mermaid
flowchart TB
    fail["attempt failed"] --> n["number of retries:<br/>request num_retries, else the deployment's num_retries,<br/>else retry_policy / model_group_retry_policy for this error type,<br/>else router num_retries (default 2)"]
    n --> should{"should_retry_this_error?"}
    should -->|"no"| raise["raise, so the fallback wrapper sees it"]
    should -->|"yes"| wait["sleep: 0 if other healthy deployments remain,<br/>else backoff honoring Retry-After, at least retry_after"]
    wait --> again["select again, skipping deployments that already failed"]
    again --> done{"success?"}
    done -->|"yes"| ok["return, with x-litellm-attempted-retries"]
    done -->|"no, retries left"| should
    done -->|"no, none left"| raise
```

`should_retry_this_error` raises at once instead of retrying in these cases. A context window error when `context_window_fallbacks` is set, and a content policy error when `content_policy_fallbacks` is set, because a fallback will do better than another deployment of the same model. A status code `litellm.should_retry` says not to retry, other than 401 and 403. A `NotFoundError`. A rate limit when no healthy deployment is left and fallbacks exist. An auth or permission error in a group with a single deployment. And any error when no healthy deployment is left. When a retry policy matched the error type, its count is used and these checks are skipped

The sleep is zero while other healthy deployments remain, since the next attempt can go somewhere else. With a single deployment, or none healthy, the router backs off with `litellm.calculate_retry_after`, which honors the provider's `Retry-After` header and the router's `retry_after` minimum

## Fallbacks

`async_function_with_fallbacks` calls the retry wrapper. When the retry wrapper gives up, `async_function_with_fallbacks_common_utils` chooses where to go next

```mermaid
flowchart TB
    e["retries exhausted, error e"] --> dis{"disable_fallbacks, or an unsupported<br/>param error with fallback_on_unsupported_params off?"}
    dis -->|"yes"| raise["raise e"]
    dis -->|"no"| ord{"several order values in the group,<br/>and e is not context window / content policy?"}
    ord -->|"yes"| ordfb["try the next order inside the same group,<br/>then the group's fallbacks"]
    ord -->|"no"| wf{"enable_weighted_failover<br/>with simple-shuffle?"}
    wf -->|"yes"| wfo["re-pick across the other deployments<br/>of the group first"]
    wf -->|"no"| kind{"what is e?"}
    wfo --> kind
    kind -->|"ContextWindowExceededError"| cw{"context_window_fallbacks set?"}
    cw -->|"yes"| cwl["use its entry for this group<br/>(no entry: raise e)"]
    cw -->|"no"| gen
    kind -->|"ContentPolicyViolationError"| cp{"content_policy_fallbacks set?"}
    cp -->|"yes"| cpl["use its entry for this group<br/>(no entry: raise e)"]
    cp -->|"no"| gen
    kind -->|"anything else"| gen["fallbacks: the group's entry,<br/>else the '*' entry (default_fallbacks)"]
    gen -->|"no entry"| raise
    cwl --> hop
    cpl --> hop
    gen --> hop
    ordfb --> hop
    hop["run_async_fallback: for each target group in order"]
```

Each list is a list of single-key dicts that map a model group to an ordered list of groups to try, for example `fallbacks=[{"gpt-4o": ["claude-sonnet", "gpt-4o-mini"]}]`. `default_fallbacks=["gpt-4o-mini"]` becomes the catch-all entry `{"*": ["gpt-4o-mini"]}`

`run_async_fallback` in `litellm/router_utils/fallback_event_handlers.py` walks the target list. For each target it checks `fallback_access_check` (may this caller use that group) and `fallback_budget_check`, skips targets this request already tried, then calls `async_function_with_fallbacks` again with the new model. So each hop gets its own full retry loop, and can fall back further. A hop counter `fallback_depth` stops the walk at `max_fallbacks` (default 5, or the `ROUTER_MAX_FALLBACKS` environment variable)

```mermaid
sequenceDiagram
    participant C as Caller
    participant F as Fallback wrapper
    participant R as Retry wrapper
    C->>F: model=A
    F->>R: group A, num_retries=2
    R-->>F: 503 after 3 attempts
    F->>F: fallbacks[A] = [B], depth 0 to 1
    F->>F: async_function_with_fallbacks(model=B)
    F->>R: group B, num_retries=2
    R-->>F: 200
    F-->>C: response from B, x-litellm-attempted-fallbacks: 1
```

With `fallbacks=[{"A": ["B"]}]`, `context_window_fallbacks=[{"A": ["A-long"]}]` and `num_retries=2`, a 503 from A is retried twice in A, then the request goes to B with two retries of its own. A context window error from A is not retried at all, since a context window fallback exists, and goes straight to A-long

Streaming has one more layer. If a stream fails after it started, `FallbackAwareStreamWrapper` can continue on a fallback group mid-stream when `enable_mid_stream_fallback_continuation` is on, combining the usage of both parts

## Cooldowns

A cooldown is a key `deployment:{id}:cooldown` in the router's cache, with a TTL equal to the cooldown time. While the key exists, the cooldown filter drops the deployment from every selection, on every pod that shares the Redis. When the TTL expires the deployment is simply eligible again. There is no probe request and no early reset on success

The failure callback, `deployment_callback_on_failure`, decides whether a failure cools the deployment down. It runs two checks, first on the error and then on the deployment's recent history

```mermaid
flowchart TB
    f["failure on deployment d"] --> skip{"cooldowns disabled, or a failure<br/>outside d (selection, caller timeout, per-user auth, ...)?"}
    skip -->|"yes"| no["no cooldown"]
    skip -->|"no"| req{"_is_cooldown_required:<br/>does this status count?"}
    req -->|"APIConnectionError,<br/>or 4xx other than 401, 402, 404, 408, 429"| no
    req -->|"429, 401, 402, 404, 408, any 5xx"| pol{"allowed_fails or a policy set<br/>(router or deployment)?"}
    pol -->|"yes"| cnt["increment deployment:d:allowed_fails<br/>(TTL = cooldown time)"]
    cnt --> over{"count > allowed fails?"}
    over -->|"yes"| cool["cool down"]
    over -->|"no"| no
    pol -->|"no"| rate{"this pod's last 60s for d"}
    rate -->|"429 and group has 2+ deployments"| cool
    rate -->|"100% failed and 1000+ requests"| cool
    rate -->|"over 50% failed, 5+ requests,<br/>group has 2+ deployments"| cool
    rate -->|"status should not be retried"| cool
    rate -->|"otherwise"| no
```

The cooldown time comes from the deployment's own `cooldown_time` if it sets one, then the provider's `Retry-After` header, then the router's `cooldown_time`, which defaults to 5 seconds (`DEFAULT_COOLDOWN_TIME_SECONDS`)

Two details are easy to miss. The success and failure counts for the error-rate rule are kept in memory only, per pod, with a 60 second TTL, so on a fleet each pod judges a deployment by the traffic it saw itself. The `allowed_fails` counter and the cooldown key itself live in Redis when it is configured, so they apply to the whole fleet. And by default a lone deployment is protected from the error-rate rule, since cooling down the only deployment of a group would fail every request for that group

When every candidate is cooled down, selection raises `RouterRateLimitError` with the time until the soonest cooldown ends. `enable_health_check_routing` changes this, by ignoring cooldowns for the request when they would leave nothing
