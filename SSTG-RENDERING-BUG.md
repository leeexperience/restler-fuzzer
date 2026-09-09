# SSTG Rendering Bug: render_with_cache Persists Invalid Prefixes Across Generations

## Problem

When using RESTler in smoke test mode with OpenAPI 3 `links` (producer-consumer dependencies), endpoints that depend on auth-protected producers are **never rendered** if the auth token isn't available in the first generation. The `render_with_cache` function caches invalid prefix renderings and skips them in subsequent generations — even when the underlying issue (auth token unavailability) has been resolved.

## Symptom

- `testing_summary.json` shows e.g. `"final_spec_coverage": "13 / 14"`
- `request_rendering.txt` shows one request "Never Rendered" with `_READER_DELIM_<var>_READER_DELIM` in its path
- The producer endpoint was failing in Gen 1-2 (auth not ready) but succeeding in Gen 3+
- Despite the producer working in later generations, the consumer is never re-attempted

## Root Cause: Cache Persistence Across Generations

### The Cache

In `driver.py:791`:
```python
seq_rendering_cache = sequences.RenderedSequenceCache()
```

This `RenderedSequenceCache` is instantiated **once** before the generation loop and persists across **all** generations.

### How Invalid Prefixes Get Cached

In `driver.py:478-479` (inside `render_with_cache`):
```python
else:
    seq_rendering_cache.add_invalid_sequence(prefix_seq_to_render)
    break
```

When a prefix of a consumer's dependency chain fails to render (e.g., because the auth token isn't ready yet), the **entire prefix** is cached as invalid.

### How the Cache Prevents Re-attempts

In `driver.py:445-447` (inside `render_with_cache`, the `get_renderings` loop):
```python
elif valid == False:
    # Do not render.
    sequences_to_render = []
```

In subsequent generations, when `render_with_cache` processes the same consumer endpoint:

1. It calls `seq_rendering_cache.get_renderings(current_seq.requests[prefix_start:])`
2. It finds the previously-cached invalid prefix (e.g., `[register, login, POST books]`)
3. `valid == False` → `sequences_to_render = []` → **rendering is skipped entirely**
4. The consumer endpoint is **never re-attempted**

### The Specific Scenario

For VAmPI's `GET /books/v1/{book_title}` (consumer of `book_title` produced by `POST /books/v1`):

- **Goal sequence**: `[POST /users/v1/register, POST /users/v1/login, POST /books/v1, GET /books/v1/{book_title}]`
- **Gen 1-2**: `POST /books/v1` fails because the auth module hasn't stabilized (JWT token not yet available). The prefix `[register, login, POST books]` is cached as invalid.
- **Gen 3+**: Auth token is now stable. `POST /books/v1` would succeed. But the cache says the prefix was invalid in Gen 1-2, so rendering is skipped.

## Code Flow (Detailed)

### 1. `driver.py:generate_sequences()` — Main fuzzing loop

```
seq_rendering_cache = RenderedSequenceCache()   # line 791 - created ONCE
for length in range(min_len, max_len):
    seq_collection = render_with_cache(...)      # line 832-834
```

### 2. `driver.py:render_with_cache()` — The renderer

```python
def render_with_cache(seq_collection, ...):
    for current_seq in seq_collection:                                # line 380
        all_rendered_prefixes_found = []
        prefix_start = 0
        valid = None
        while prefix_start < current_seq.length - 1:                  # line 384
            renderings_found = seq_rendering_cache.get_renderings(
                current_seq.requests[prefix_start:], ...)             # line 385

            if renderings_found:
                if True in renderings_found:
                    valid = True
                else:
                    valid = False                                     # line 391

                if valid == False:
                    # STOP - don't render, don't continue             # line 445-447
                    sequences_to_render = []
                    break                                             # line 427
            else:
                break                                                 # line 429

        # Now render prefixes not yet in cache
        for prefix_len in range(rendered_prefix_length + 1, sequence_to_render.length + 1):
            valid_renderings = render_one(prefix_seq_to_render, ...)  # line 460
            if valid_renderings:
                seq_rendering_cache.add_valid_prefixes(valid_rendering)  # line 465
            else:
                seq_rendering_cache.add_invalid_sequence(prefix_seq_to_render)  # line 479
                break  # STOP - don't try longer prefixes             # line 490
```

### 3. `sequences.py:RenderedSequenceCache` — The cache

```python
class RenderedSequenceCache(object):
    def add_invalid_sequence(self, sequence):                         # line 959
        self.add(sequence, False)                                     # line 972

    def get_renderings(self, req_list, ...):                          # line 1006
        # Searches for longest matching prefix
        for prefix_len in range(len(req_list), 0, -1):
            # Checks cache for this prefix
            ...
```

## The Specific Failing Endpoint

From the actual `request_rendering.txt` (run `20260626_114956_0.5h`):

```
Never Rendered requests:
    Request: 13
        - restler_static_string: 'GET '
        - restler_static_string: '/books/v1/'
        - restler_static_string: '_READER_DELIM_books_v1_post_book_title_READER_DELIM'
        - restler_static_string: ' HTTP/1.1\r\n'
        + restler_refreshable_authentication_token: [...]
```

The `_READER_DELIM` in the URL path means the dynamic variable was never resolved — the producer (`POST /books/v1`) never created the `book_title` value in a sequence that was considered valid by the cache.

## How to Reproduce

1. Use a RESTler-compiled grammar with OpenAPI 3 `links` producing auth-dependent dynamic variables
2. Run RESTler in smoke test mode (`--mode directed-smoke-test`)
3. Configure auth via module (not token file), so auth isn't ready in Gen 1
4. Observe: consumer endpoints of auth-protected producers are never rendered

## Potential Fix Approaches

### Approach 1: Per-generation cache invalidation
Reset the `seq_rendering_cache` each generation (move instantiation inside the generation loop). **Risk:** Performance regression from re-rendering valid prefixes, and potential for infinite retry loops on truly-invalid prefixes. **Best if combined with** a limit on re-attempts.

### Approach 2: Re-attempt invalid prefixes with backoff
When `valid == False`, don't skip; instead, allow re-attempting up to N times. Add a retry counter to the cache. This handles transient failures (auth not ready) without infinite retries.

### Approach 3: Auth-aware rendering priority
Pre-render auth-producing sequences first (in preprocessing), so the auth token pool is populated before any consumer sequences are processed. This is a broader change that requires understanding RESTler's auth flow.

### Approach 4: Selective cache clearing
Clear only the `add_invalid_sequence` entries each generation, preserving `add_valid_prefixes`. This keeps valid prefix performance gains while allowing re-attempts on failures.

## Key Source Files and Line Numbers

- `restler/engine/core/driver.py:347-497` — `render_with_cache` function (the core problem)
- `restler/engine/core/driver.py:791` — Cache instantiation (persists across generations)
- `restler/engine/core/driver.py:445-447` — Invalid prefix skip (the `valid == False` branch)
- `restler/engine/core/driver.py:478-479` — Invalid prefix caching
- `restler/engine/core/sequences.py:886-1024` — `RenderedSequenceCache` class
- `restler/engine/core/sequences.py:959-972` — `add_invalid_sequence` method
- `restler/engine/core/sequences.py:1006-1024` — `get_renderings` method (cache lookup)

## Context

This was discovered during the SSTG (Semantic State-Transition Graph) thesis project at https://github.com/anomalyco/opencode. The project injects OpenAPI 3 `links` into the spec to guide RESTler to follow a custom state graph. The `$request.body` fix for RESTler's compiler is at `src/compiler/Restler.Compiler/Annotations.fs:252` — the engine-side rendering issue is the remaining blocker.
