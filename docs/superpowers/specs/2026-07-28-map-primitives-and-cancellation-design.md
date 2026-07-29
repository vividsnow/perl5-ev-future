# EV::Future: map primitives and cancellation handles

Date: 2026-07-28
Status: approved, not yet implemented
Target release: 0.07 (0.06 is the unsafe-exception abandonment fix)

## Motivation

Every example under `eg/` has the same shape:

```perl
parallel_limit([
    map {
        my $url = $_;
        sub {
            my $done = shift;
            $client->request($url, sub { my ($resp, $err) = @_; ...; $done->() });
        }
    } @urls
], 2, sub { ... });
```

Two costs fall out of it. The call site allocates one closure per item purely to
bind the item, and there is no way to stop the operation once it is running.
`parallel_limit`'s POD states the second outright: "There is no cancellation
mechanism; all dispatched tasks must complete."

This spec addresses both. It is deliberately additive: nothing here changes the
meaning of `done`'s arguments, which remain inconsistent across the four
existing primitives (ignored by `parallel` and `parallel_limit`, truthy-first-arg
cancels in `series`, forwarded to `final_cb` by `race`). Resolving that
collision is a prerequisite for an error or result channel and is out of scope.

## Scope

In scope:

- `parallel_map`, `parallel_map_limit`, `series_map`
- a cancellation handle with `cancel`, `pending` and `active`, returned by all
  seven primitives

Explicitly out of scope:

- `race_map` (no demonstrated need)
- node-style `done->($err, @value)` and ordered result collection
- `waterfall`, `timeout`, `retry`
- reusable or precompiled plan objects

## Design

### 1. Map primitives

```
parallel_map      (\@items, \&worker,         \&final_cb, [$unsafe])
parallel_map_limit(\@items, \&worker, $limit, \&final_cb, [$unsafe])
series_map        (\@items, \&worker,         \&final_cb, [$unsafe])
```

Each mirrors the positional signature of its non-map counterpart, so `$limit`
stays before `final_cb` and `$unsafe` stays last.

The worker receives `($item, $done)`:

```perl
parallel_map_limit(\@urls, sub {
    my ($url, $done) = @_;
    $client->request($url, sub { my ($resp, $err) = @_; ...; $done->() });
}, 4, sub { say "all fetched" });
```

Item first reads naturally as `my ($item, $done) = @_` and leaves `$index`
appendable as a third argument later without breaking callers, which matters if
the result channel ever lands.

#### Implementation

These reuse the existing contexts and dispatch loops rather than duplicating
them. Each of `parallel_ctx`, `plimit_ctx` and `series_ctx` gains one field:

```c
SV *worker;  /* NULL for the task form */
```

Every dispatch site currently does:

```c
PUSHMARK(SP); XPUSHs(done_rv); PUTBACK;
call_sv(task_sv, flags);
```

With `worker` set it pushes `(item, done_rv)` and calls `worker` instead of the
array element. That is one field and one branch per dispatch site. All four
cleanup functions, the `running`/`delayed` trampoline, the `ifp_guard` chain
added in 0.06, and the `AvREAL`/`SvMAGICAL` fast path with its `av_fetch`
fallback carry over unchanged.

The `worker` SV is refcount-incremented into the context at construction and
decremented in the matching `*_cleanup`, exactly as `final_cb` already is.

#### Semantic divergences from the task form

- **Non-coderef elements are data, not no-ops.** The task form treats a
  non-coderef element as a task that completes instantly. The map form must not:
  `\@items` holds values, so every element is dispatched to the worker
  regardless of type. This is a real behavioural fork between the two forms and
  must be documented.
- **A non-coderef worker croaks up front**, before any dispatch and before the
  context is allocated. That is a programming error, unlike a degenerate task
  list. Note that `IS_PVCV` only accepts a genuine `SVt_PVCV`, so an object with
  a `&{}` overload croaks here; that limitation is already documented for tasks.
- **Items are passed as `sv_2mortal(SvREFCNT_inc(item))`**, not
  `sv_mortalcopy(item)`. Cheaper than copying, safe if the worker mutates
  `\@items` during its own call, and it gives the same aliasing semantics as
  Perl's own `for` and `map`.

Unsafe mode behaves exactly as it does for the task form: a single shared `done`
CV, no `G_EVAL`, and an escaping exception abandons the operation per the 0.06
contract.

### 2. Cancellation handle

#### Allocation

The handle is built only in non-void context. Each XSUB checks `GIMME_V` and
skips the allocation entirely otherwise, so every existing call site, and
`bench/`, pays nothing.

#### Representation

The handle cannot hold a raw context pointer: the context is freed the moment the
operation completes, and the caller may hold the handle past that point. A small
refcounted cell sits between them:

```c
typedef struct {
    void *ctx;      /* NULL once the operation is over */
    int   kind;     /* which primitive, to pick the right cleanup */
    int   refcnt;   /* held by the context and by the Perl object */
} evf_handle;
```

The context gains `evf_handle *h` (NULL when the caller did not ask for one).
Each `*_cleanup` does `if (ctx->h) { ctx->h->ctx = NULL; evf_handle_dec(ctx->h); }`.
The Perl-side `DESTROY` calls `evf_handle_dec`. Whichever side drops last frees
the cell. With `ctx` NULLed on completion, every later method call is a safe
no-op.

The Perl object is a blessed scalar ref holding the cell pointer, in package
`EV::Future::Handle`.

#### `$h->cancel([$fire])`

- Context already gone, or already cancelled: no-op. Idempotent by construction.
- Otherwise run the primitive's normal `*_cleanup`. That NULLs `any_ptr` on every
  outstanding `done` CV, so in-flight tasks calling `done` later are ignored, and
  nothing leaks.
- If `$fire` is true, invoke `final_cb` once with no arguments afterwards, on the
  same path normal completion uses.

Bare `cancel` abandons silently; `cancel(1)` means "I have what I need, wrap up".
This mirrors `series`' existing truthy-argument convention and needs no new
argument channel.

Cancelling from inside a task already works through the existing machinery:
`*_cleanup` sets `*(ctx->is_freed_ptr) = 1`, and each dispatch loop's
`if (*is_freed) goto done` sees it on the next turn. Cancelling from `final_cb`
is a no-op, since cleanup has already run by then.

For `series` this is complementary rather than redundant: truthy-`done` cancels
from inside a task, the handle cancels from outside one.

#### `$h->pending` and `$h->active`

`pending` is tasks not yet completed, counting both in-flight and not yet
dispatched. `active` is tasks currently in flight. Both come from fields that
already exist:

| primitive | `active` | `pending` |
|---|---|---|
| `parallel`, `parallel_map` | `ctx->remaining` | `ctx->remaining` |
| `parallel_limit`, `parallel_map_limit` | `ctx->active` | `ctx->remaining` |
| `series`, `series_map` | `1` | `total_tasks - current_idx + 1` |
| `race` | `total_tasks` | `total_tasks` |

The only structural change is one new `I32 total_tasks` in `race_ctx`, assigned
once at construction. There is no new per-task bookkeeping on any dispatch path,
so the hot loops and the benchmark are unaffected.

`parallel` and `race` collapse the two counts because they dispatch everything up
front, and the handle is not returned until the XSUB returns, by which point
nothing undispatched remains to observe. `series` reports `active == 1` for its
whole life, which is degenerate but correct.

The `series` formula holds only over the window in which the handle is
observable. `current_idx` is incremented before each task is called, so it is
already at least 1 by the time the XSUB returns; the formula would read one too
high before the first dispatch, which no caller can reach. Truthy-`done`
cancellation sets `current_idx = total_tasks`, so the count is clamped at 0 to
stay correct if it is read from inside the cancelling task, before cleanup
runs.

Both return `0` once the operation is over. This gives an invariant worth
documenting: **`pending == 0` if and only if the operation has finished or been
cancelled**, which removes any need for a separate completion predicate.

### 3. Interaction with existing behaviour

- `series`' truthy-`done` cancellation is unaffected and remains the in-task way
  to cancel.
- Unsafe mode's documented double-call hazard corrupts `remaining`, so a
  double-called `done` makes `pending` under-report for `parallel` and
  `parallel_limit`. Same root cause as premature `final_cb`; document both
  together.
- **Behaviour change:** `my $x = parallel(...)` currently yields undef and will
  start yielding a handle. The functions are documented as returning nothing, so
  the risk is low, but it is a change and belongs in `Changes`.

## Testing

Map primitives:

- item delivery and worker argument order
- `series_map` ordering; `parallel_map_limit` respecting `$limit`
- empty `\@items` fires `final_cb` immediately
- holey, tied and magic arrays, and mutation mid-dispatch, reusing the patterns
  in `t/01-basic.t` and `t/03-race.t`
- non-coderef items dispatched as data, not skipped as no-ops
- non-coderef worker croaks before any dispatch
- unsafe-mode parity, including exception abandonment per 0.06
- nesting a map primitive inside another primitive

Handle:

- void context allocates nothing
- `cancel` before any dispatch, mid-flight from a timer, from inside a task, from
  `final_cb`, twice, and after completion
- `cancel(1)` fires `final_cb` exactly once; bare `cancel` never fires it
- handle outliving the operation, and the operation outliving the handle, in both
  `DESTROY` orders
- `pending` and `active` through a full lifecycle for each primitive: after
  construction, mid-flight, inside a task, immediately before completion, after
  completion, and after cancel
- `parallel_limit` asserting `active <= limit` and `active < pending` while tasks
  are queued, which is the one case where the two genuinely diverge

All of the above must pass under the existing CI gates: ASan plus UBSan with
`detect_stack_use_after_return=1`, and valgrind with
`--errors-for-leak-kinds=definite`.

## Documentation

- POD for the three new functions and for `EV::Future::Handle`
- POD note that non-coderef items are data in the map form
- POD note on `pending` under-reporting after an unsafe double-call
- README function list
- `Changes` entry, including the scalar-context return change
- new test files added to `MANIFEST`

## Risks

- The map form's "non-coderef elements are data" rule is the opposite of the task
  form's rule. Two adjacent functions with opposite behaviour for the same input
  shape is a documentation burden and a plausible source of user confusion.
- The handle adds a second lifetime to reason about alongside the context and the
  `ifp_guard` chain. The refcounted cell keeps them independent, but the
  `DESTROY`-ordering tests are load-bearing rather than incidental.
- `GIMME_V`-conditional allocation means the handle path is not exercised by any
  existing call site, so it needs its own coverage rather than riding along on
  the current suite.
