# Map Primitives and Cancellation Handles Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `parallel_map`, `parallel_map_limit` and `series_map`, plus a cancellation handle with `cancel`, `pending` and `active` returned by all seven primitives.

**Architecture:** The three map primitives reuse the existing contexts and dispatch loops rather than duplicating them; each context gains one `SV *worker` field and each dispatch site gains one branch that pushes `(item, done)` and calls the worker instead of the array element. The handle is a refcounted cell sitting between the context and a blessed Perl scalar, allocated only in non-void context so existing call sites pay nothing.

**Tech Stack:** Perl XS (xsubpp), libev via the EV module, ExtUtils::MakeMaker, Test::More.

Spec: `docs/superpowers/specs/2026-07-28-map-primitives-and-cancellation-design.md`

## Global Constraints

- Perl 5.10.0 minimum (`MIN_PERL_VERSION => '5.010000'`); no post-5.10 syntax.
- EV 4.37 minimum.
- **Build and test with `perl Makefile.PL && make` then `prove -Iblib/lib -Iblib/arch t/`.** Never use `prove -l t/`: `lib/` has no `auto/` directory, so `XSLoader` falls through `@INC` and silently loads the *installed* `Future.so` from site_perl instead of your build. A green `prove -l` run proves nothing.
- No em dashes in POD or documentation; use semicolons or hyphens.
- No AI attribution in commit messages.
- Target version 0.07 (0.06 is the shipped unsafe-exception abandonment fix).
- Every new test file must be added to `MANIFEST`.
- ASan+UBSan and valgrind must stay clean; commands are in Task 8 and must be run before the final commit.
- **Valgrind must be pointed at the real perl binary, not `perl` on `PATH`.** On this machine `perl` is a plenv shim, which is a bash script that `execve`s the real interpreter. Valgrind does not follow an `exec` unless `--trace-children=yes` is passed, so `valgrind ... perl t/foo.t` silently instruments nothing: it prints no `ERROR SUMMARY` and exits with the test suite's own status, which looks exactly like a clean run. Use `"$(perl -MConfig -e 'print $Config{perlpath}')"` and **confirm an `ERROR SUMMARY` line is actually present in the output**; a bare exit code is not evidence. ASan is unaffected, because `LD_PRELOAD` and compile-time instrumentation both survive `exec`.
- `IS_PVCV(sv)` is the existing macro at `Future.xs:8`; it accepts only a genuine `SVt_PVCV`, so `&{}`-overloaded objects are not code refs.

**Verification commands used throughout:**

```bash
# Build
perl Makefile.PL && make

# Full suite (correct invocation)
prove -Iblib/lib -Iblib/arch t/

# Single test file, verbose
perl -Mblib -Iblib/arch -Iblib/lib t/04-map.t
```

---

## File Structure

| File | Responsibility | Change |
|---|---|---|
| `Future.xs` | All XS: contexts, dispatch, cleanup, XSUBs | Modify |
| `lib/EV/Future.pm` | `@EXPORT`, `$VERSION`, POD | Modify |
| `t/04-map.t` | Map primitive behaviour | Create |
| `t/05-handle.t` | Handle lifetime, cancel, pending, active | Create |
| `MANIFEST` | Dist file list | Modify |
| `README.md` | Function list | Modify |
| `Changes` | Release notes | Modify |

`Future.xs` stays a single file. It is 900 lines and cohesive; splitting it would mean juggling headers for shared static functions and structs for no benefit.

---

### Task 1: Extract the three dispatch bodies into static start functions

Pure refactor, no behaviour change. This exists so Tasks 2 to 4 can add a `worker` parameter in one place per primitive instead of duplicating a 100-line XSUB body. A reviewer can accept or reject this independently: every existing test must pass unchanged.

**Files:**
- Modify: `Future.xs` (XSUB bodies at `Future.xs:582` `parallel`, `Future.xs:701` `series`, `Future.xs:747` `parallel_limit`)

**Interfaces:**
- Consumes: nothing
- Produces:
  - `static void parallel_start(pTHX_ AV *list, SV *final_cb, int unsafe)`
  - `static void series_start(pTHX_ AV *list, SV *final_cb, int unsafe)`
  - `static void plimit_start(pTHX_ AV *list, I32 limit, SV *final_cb, int unsafe)`

  `race` is deliberately **not** extracted; there is no `race_map`, and its handle wiring in Task 5 happens inline in its XSUB.

- [ ] **Step 1: Run the existing suite to establish a green baseline**

```bash
perl Makefile.PL && make && prove -Iblib/lib -Iblib/arch t/
```

Expected: `All tests successful. Files=9, Tests=54. Result: PASS`

- [ ] **Step 2: Add the three forward declarations**

Insert immediately after the `race_ctx` typedef ends at `Future.xs:110`:

```c
static void parallel_start(pTHX_ AV *list, SV *final_cb, int unsafe);
static void series_start(pTHX_ AV *list, SV *final_cb, int unsafe);
static void plimit_start(pTHX_ AV *list, I32 limit, SV *final_cb, int unsafe);
```

- [ ] **Step 3: Extract `parallel_start`**

Add this function immediately before the `MODULE = EV::Future` line at `Future.xs:785`. The body is the current `parallel` XSUB `CODE:` block moved verbatim, with three mechanical changes: `tasks` becomes `list`, the `unsafe` computation is removed (it is now a parameter), and `return;` in the empty-list branch stays as-is.

```c
static void parallel_start(pTHX_ AV *list, SV *final_cb, int unsafe) {
    I32 len = av_len(list) + 1;
    if (len <= 0) {
        if (IS_PVCV(final_cb)) {
            dSP;
            ENTER;
            SAVETMPS;
            PUSHMARK(SP);
            PUTBACK;
            call_sv(final_cb, G_DISCARD | G_VOID);
            FREETMPS;
            LEAVE;
        }
        return;
    }

    /* Move `Future.xs:591-699` here verbatim: the current parallel XSUB body
       from `ENTER;` down to and including the final `LEAVE;` at line 699.
       Every occurrence of the identifier `tasks` becomes `list`. Change
       nothing else. */
}
```

The body is moved rather than reproduced here on purpose: it is 100 lines of
working, sanitizer-clean code already in the repo, and retyping it invites
transcription errors. Use your editor's cut and paste, not the keyboard.

The corresponding ranges for the other two extractions in Step 5 are
`Future.xs:707-745` for `series` and `Future.xs:753-808` for `parallel_limit`.
Verify the ranges before cutting; earlier steps in this task do not shift them,
but a partially applied edit would.

- [ ] **Step 4: Replace the `parallel` XSUB body**

```c
void
parallel(list, final_cb, ...)
    AV *list
    SV *final_cb
    CODE:
        int unsafe = (items > 2 && SvTRUE(ST(2)));
        parallel_start(aTHX_ list, final_cb, unsafe);
```

**Critical:** the first parameter must be named `list`, not `tasks` and never `items`. `items` is the argument-count variable xsubpp generates for a `...` XSUB, and the existing code reads it in `items > 2 && SvTRUE(ST(2))`. Naming a parameter `items` shadows it and silently breaks `$unsafe` detection.

- [ ] **Step 5: Extract `series_start` and `plimit_start` the same way**

`series_start` takes the current `series` XSUB body verbatim; `plimit_start` takes the current `parallel_limit` body verbatim including the `if (limit < 1) limit = 1; if (limit > len) limit = len;` clamp. Rename `tasks` to `list` in both. Then:

```c
void
series(list, final_cb, ...)
    AV *list
    SV *final_cb
    CODE:
        int unsafe = (items > 2 && SvTRUE(ST(2)));
        series_start(aTHX_ list, final_cb, unsafe);

void
parallel_limit(list, limit, final_cb, ...)
    AV *list
    I32 limit
    SV *final_cb
    CODE:
        int unsafe = (items > 3 && SvTRUE(ST(3)));
        plimit_start(aTHX_ list, limit, final_cb, unsafe);
```

- [ ] **Step 6: Rebuild and verify the suite is still green**

```bash
make && prove -Iblib/lib -Iblib/arch t/
```

Expected: `All tests successful. Files=9, Tests=54. Result: PASS`, identical to Step 1. Any change in test count or any failure means the extraction was not verbatim.

- [ ] **Step 7: Commit**

```bash
git add Future.xs
git commit -m "refactor: extract parallel/series/parallel_limit bodies into static start functions"
```

---

### Task 2: `parallel_map`

**Files:**
- Modify: `Future.xs` (`parallel_ctx` at `Future.xs:10-19`, `parallel_cleanup` at `Future.xs:116`, `parallel_start`, new XSUB)
- Modify: `lib/EV/Future.pm` (`@EXPORT`, POD)
- Create: `t/04-map.t`
- Modify: `MANIFEST`

**Interfaces:**
- Consumes: `parallel_start` from Task 1
- Produces:
  - `parallel_ctx.worker` (`SV *`, NULL for the task form)
  - `static void parallel_start(pTHX_ AV *list, SV *worker, SV *final_cb, int unsafe)` (signature changed: `worker` added as second parameter)
  - Perl function `parallel_map(\@items, \&worker, \&final_cb, [$unsafe])`

- [ ] **Step 1: Write the failing test**

Create `t/04-map.t`:

```perl
use strict;
use warnings;
use Test::More;
use EV;
use EV::Future;

subtest 'parallel_map passes each item to the worker' => sub {
    my @seen;
    my $final = 0;
    parallel_map([qw(a b c)], sub {
        my ($item, $done) = @_;
        push @seen, $item;
        $done->();
    }, sub { $final++ });

    is_deeply([sort @seen], [qw(a b c)], 'worker saw every item');
    is($final, 1, 'final_cb fired once');
};

subtest 'parallel_map worker gets item first, done second' => sub {
    my ($got_item, $got_done);
    parallel_map(['only'], sub {
        ($got_item, $got_done) = @_;
        $got_done->();
    }, sub { });

    is($got_item, 'only', 'first argument is the item');
    is(ref($got_done), 'CODE', 'second argument is the done callback');
};

subtest 'parallel_map treats non-coderef items as data' => sub {
    my @seen;
    my $final = 0;
    parallel_map([1, undef, 'x'], sub {
        my ($item, $done) = @_;
        push @seen, defined $item ? $item : '(undef)';
        $done->();
    }, sub { $final++ });

    is(scalar @seen, 3, 'all three items dispatched, none skipped as no-ops');
    is($final, 1, 'final_cb fired once');
};

subtest 'parallel_map croaks on a non-coderef worker' => sub {
    eval { parallel_map([1, 2], 'not a coderef', sub { }) };
    like($@, qr/worker must be a code reference/, 'croaks before dispatch');
};

subtest 'parallel_map with an empty list fires final_cb immediately' => sub {
    my $final = 0;
    parallel_map([], sub { die 'never called' }, sub { $final++ });
    is($final, 1, 'final_cb fired');
};

subtest 'parallel_map async' => sub {
    our @w;
    my @seen;
    my $final = 0;
    parallel_map([qw(x y z)], sub {
        my ($item, $done) = @_;
        push @w, EV::timer 0.01, 0, sub { push @seen, $item; $done->() };
    }, sub { $final++; EV::break });

    EV::run;
    is(scalar @seen, 3, 'all async items completed');
    is($final, 1, 'final_cb fired once');
    @w = ();
};

subtest 'parallel_map unsafe mode' => sub {
    my @seen;
    my $final = 0;
    parallel_map([qw(a b)], sub {
        my ($item, $done) = @_;
        push @seen, $item;
        $done->();
    }, sub { $final++ }, 1);

    is(scalar @seen, 2, 'unsafe mode dispatched both items');
    is($final, 1, 'final_cb fired once');
};

subtest 'parallel_map with a holey array passes undef' => sub {
    my $list = [];
    $list->[2] = 'third';
    my @seen;
    parallel_map($list, sub {
        my ($item, $done) = @_;
        push @seen, defined $item ? $item : '(undef)';
        $done->();
    }, sub { });

    is_deeply(\@seen, ['(undef)', '(undef)', 'third'], 'holes arrive as undef');
};

done_testing;
```

- [ ] **Step 2: Run it to verify it fails**

```bash
perl -Mblib -Iblib/arch -Iblib/lib t/04-map.t
```

Expected: FAIL with `Undefined subroutine &main::parallel_map called`.

- [ ] **Step 3: Add the `worker` field to `parallel_ctx`**

In the struct at `Future.xs:10-19`, add after `SV *final_cb;`:

```c
    SV *worker;    /* NULL for the task form; the worker CV for the map form */
```

- [ ] **Step 4: Release `worker` in `parallel_cleanup`**

In `parallel_cleanup` (`Future.xs:116`), immediately after the existing `if (ctx->final_cb) SvREFCNT_dec(ctx->final_cb);`:

```c
    if (ctx->worker) SvREFCNT_dec(ctx->worker);
```

- [ ] **Step 5: Add the `worker` parameter to `parallel_start`**

Change the signature and the forward declaration to:

```c
static void parallel_start(pTHX_ AV *list, SV *worker, SV *final_cb, int unsafe)
```

Initialise the field where the other context fields are set:

```c
    ctx->worker = worker ? SvREFCNT_inc(worker) : NULL;
```

Then change the dispatch branch inside the `for (i = 0; i < len; i++)` loop. It currently reads `if (IS_PVCV(task_sv)) {`. Replace that condition and the two argument-push lines:

```c
            if (ctx->worker || IS_PVCV(task_sv)) {
                CV *cv = NULL;
                if (!unsafe) {
                    cv = newXS(NULL, parallel_task_done, __FILE__);
                    CvXSUBANY(cv).any_ptr = ctx;
                    ctx->cvs[i] = (CV*)SvREFCNT_inc((SV*)cv);
                }

                ENTER;
                SAVETMPS;
                if (!unsafe) {
                    done_rv = sv_2mortal(newRV_noinc((SV*)cv));
                }
                PUSHMARK(SP);
                if (ctx->worker) {
                    XPUSHs(task_sv ? sv_2mortal(SvREFCNT_inc(task_sv)) : &PL_sv_undef);
                }
                XPUSHs(done_rv);
                PUTBACK;

                call_sv(ctx->worker ? ctx->worker : task_sv, flags);
```

Leave the rest of the branch (the `SPAGAIN` / `SvTRUE(ERRSV)` error handling, `FREETMPS`, `LEAVE`) exactly as it is.

The item is passed as `sv_2mortal(SvREFCNT_inc(task_sv))`, not `sv_mortalcopy`: it is cheaper than copying, it stays alive if the worker mutates the array during its own call, and it gives the same aliasing semantics as Perl's `for` and `map`. A hole (`av_fetch` returned NULL) arrives as `undef`.

- [ ] **Step 6: Update the `parallel` XSUB call site**

```c
        parallel_start(aTHX_ list, NULL, final_cb, unsafe);
```

- [ ] **Step 7: Add the `parallel_map` XSUB**

Immediately after the `parallel` XSUB:

```c
void
parallel_map(list, worker, final_cb, ...)
    AV *list
    SV *worker
    SV *final_cb
    CODE:
        int unsafe = (items > 3 && SvTRUE(ST(3)));
        if (!IS_PVCV(worker))
            croak("EV::Future::parallel_map: worker must be a code reference");
        parallel_start(aTHX_ list, worker, final_cb, unsafe);
```

The croak comes before any dispatch and before the context is allocated, so there is nothing to clean up.

- [ ] **Step 8: Export it**

In `lib/EV/Future.pm`, change the `@EXPORT` line to:

```perl
our @EXPORT = qw(parallel parallel_limit series race parallel_map);
```

- [ ] **Step 9: Run the test to verify it passes**

```bash
make && perl -Mblib -Iblib/arch -Iblib/lib t/04-map.t
```

Expected: PASS, 8 subtests.

- [ ] **Step 10: Run the full suite**

```bash
prove -Iblib/lib -Iblib/arch t/
```

Expected: `All tests successful.` with `t/04-map.t` included.

- [ ] **Step 11: Add POD for `parallel_map`**

In `lib/EV/Future.pm`, after the existing `=head2 parallel(\@tasks, \&final_cb, [$unsafe])` section:

```pod
=head2 parallel_map(\@items, \&worker, \&final_cb, [$unsafe])

Dispatch C<\&worker> once per element of C<\@items>, passing
C<($item, $done)>; call C<final_cb> once every worker has invoked its C<done>.

  parallel_map(\@urls, sub {
      my ($url, $done) = @_;
      $client->request($url, sub { ...; $done->() });
  }, sub { print "all fetched\n" });

Unlike the task form, elements are data: a non-coderef element is dispatched to
the worker like any other value, not treated as a task that completes
instantly. Holes in C<\@items> arrive as C<undef>. A non-coderef C<\&worker>
is a fatal error, raised before any dispatch.

The item is passed without copying, so modifying C<$_[0]> modifies the array
element, as with Perl's own C<for> and C<map>.
```

- [ ] **Step 12: Add the test file to MANIFEST**

Insert `t/04-map.t` into `MANIFEST` in sorted position, after `t/03-race.t`.

- [ ] **Step 13: Verify POD and MANIFEST**

```bash
podchecker lib/EV/Future.pm
perl -MExtUtils::Manifest=manifind,maniskip -e 'my $s=maniskip(); my $f=manifind(); open my $m,"<","MANIFEST"; my %i = map { (split /\s+/)[0] => 1 } <$m>; chomp %i; my @x = grep { !$s->($_) && !$i{$_} } sort keys %$f; print @x ? "UNLISTED: @x\n" : "MANIFEST clean\n";'
```

Expected: `lib/EV/Future.pm pod syntax OK.` and `MANIFEST clean`.

- [ ] **Step 14: Commit**

```bash
git add Future.xs lib/EV/Future.pm t/04-map.t MANIFEST
git commit -m "feat: add parallel_map"
```

---

### Task 3: `parallel_map_limit`

**Files:**
- Modify: `Future.xs` (`plimit_ctx` at `Future.xs:83-99`, `plimit_cleanup` at `Future.xs:331`, `_plimit_dispatch` at `Future.xs:402`, `plimit_start`, new XSUB)
- Modify: `lib/EV/Future.pm` (`@EXPORT`, POD)
- Modify: `t/04-map.t`

**Interfaces:**
- Consumes: `plimit_start` from Task 1
- Produces:
  - `plimit_ctx.worker` (`SV *`)
  - `static void plimit_start(pTHX_ AV *list, SV *worker, I32 limit, SV *final_cb, int unsafe)`
  - Perl function `parallel_map_limit(\@items, \&worker, $limit, \&final_cb, [$unsafe])`

- [ ] **Step 1: Write the failing test**

Append to `t/04-map.t`, before `done_testing;`:

```perl
subtest 'parallel_map_limit respects the limit' => sub {
    our @w;
    my ($active, $max_active, $final) = (0, 0, 0);
    my @seen;

    parallel_map_limit([1 .. 6], sub {
        my ($item, $done) = @_;
        $active++;
        $max_active = $active if $active > $max_active;
        push @w, EV::timer 0.01, 0, sub {
            push @seen, $item;
            $active--;
            $done->();
        };
    }, 2, sub { $final++; EV::break });

    EV::run;
    is(scalar @seen, 6, 'all six items completed');
    cmp_ok($max_active, '<=', 2, 'never more than 2 in flight');
    is($final, 1, 'final_cb fired once');
    @w = ();
};

subtest 'parallel_map_limit croaks on a non-coderef worker' => sub {
    eval { parallel_map_limit([1, 2], undef, 2, sub { }) };
    like($@, qr/worker must be a code reference/, 'croaks before dispatch');
};

subtest 'parallel_map_limit unsafe mode' => sub {
    my @seen;
    my $final = 0;
    parallel_map_limit([qw(a b c)], sub {
        my ($item, $done) = @_;
        push @seen, $item;
        $done->();
    }, 2, sub { $final++ }, 1);

    is(scalar @seen, 3, 'unsafe mode dispatched every item');
    is($final, 1, 'final_cb fired once');
};
```

- [ ] **Step 2: Run it to verify it fails**

```bash
perl -Mblib -Iblib/arch -Iblib/lib t/04-map.t
```

Expected: FAIL with `Undefined subroutine &main::parallel_map_limit called`.

- [ ] **Step 3: Add the `worker` field to `plimit_ctx`**

In the struct at `Future.xs:83-99`, after `SV *final_cb;`:

```c
    SV *worker;    /* NULL for the task form; the worker CV for the map form */
```

- [ ] **Step 4: Release it in `plimit_cleanup`**

In `plimit_cleanup` (`Future.xs:331`), after `if (ctx->final_cb) SvREFCNT_dec(ctx->final_cb);`:

```c
    if (ctx->worker) SvREFCNT_dec(ctx->worker);
```

- [ ] **Step 5: Add the `worker` parameter to `plimit_start`**

Change the signature and forward declaration to:

```c
static void plimit_start(pTHX_ AV *list, SV *worker, I32 limit, SV *final_cb, int unsafe)
```

and initialise:

```c
    ctx->worker = worker ? SvREFCNT_inc(worker) : NULL;
```

Update the `parallel_limit` XSUB call site to pass `NULL`:

```c
        plimit_start(aTHX_ list, NULL, limit, final_cb, unsafe);
```

- [ ] **Step 6: Add the dispatch branch in `_plimit_dispatch`**

In `_plimit_dispatch` (`Future.xs:402`), first hoist the worker out of the loop.
Immediately after the guard is installed and before `ctx->running = 1;`, add:

```c
    SV *worker = ctx->worker;
    const int have_worker = (worker != NULL);
```

This is not cosmetic. `av_fetch` on a magical (tied) array runs the tie's
`FETCH`, which is arbitrary Perl; in unsafe mode that Perl can call an
already-handed-out `$done` enough times to drive `remaining` to 0, which frees
the context from underneath the loop. Any `ctx->` dereference between the
`*is_freed` check and `call_sv` is therefore a read-after-free. Task 2 shipped
exactly this defect and had to fix it in review; do not reintroduce it here.
`ctx->worker` never changes after construction, so the hoist is
behaviour-preserving.

Then the inner loop, which currently reads `if (IS_PVCV(task_sv)) {`. Change the
condition and the argument push:

```c
            if (have_worker || IS_PVCV(task_sv)) {
                SV *done_rv = NULL;
                CV *cv = NULL;
                dSP;

                if (!ctx->unsafe) {
                    cv = newXS(NULL, plimit_task_done, __FILE__);
                    CvXSUBANY(cv).any_ptr = ctx;
                    ctx->cvs[ctx->current_idx - 1] = (CV*)SvREFCNT_inc((SV*)cv);
                }

                ctx->active++;
                int task_unsafe = ctx->unsafe;

                ENTER;
                SAVETMPS;
                if (!ctx->unsafe) {
                    done_rv = sv_2mortal(newRV_noinc((SV*)cv));
                } else {
                    done_rv = sv_2mortal(newRV_inc((SV*)ctx->shared_cv));
                }
                PUSHMARK(SP);
                if (have_worker) {
                    XPUSHs(task_sv ? sv_2mortal(SvREFCNT_inc(task_sv)) : &PL_sv_undef);
                }
                XPUSHs(done_rv);
                PUTBACK;

                U32 flags = G_DISCARD | (task_unsafe ? 0 : G_EVAL);
                call_sv(have_worker ? worker : task_sv, flags);
```

Leave everything below `call_sv` in that branch unchanged, including the `if (!task_unsafe)` error handling, `FREETMPS`, `LEAVE`, and `if (*is_freed) goto done;`.

Note that `_plimit_dispatch` already dereferences `ctx` inside this window in
code you are not changing (`ctx->current_idx++`, `ctx->active++`, `ctx->unsafe`).
That is a pre-existing condition, tracked separately; do not attempt to fix it
in this task. Your obligation is only that you add no new `ctx->` access
between the `*is_freed` check and `call_sv`.

- [ ] **Step 7: Add the `parallel_map_limit` XSUB**

Immediately after the `parallel_limit` XSUB:

```c
void
parallel_map_limit(list, worker, limit, final_cb, ...)
    AV *list
    SV *worker
    I32 limit
    SV *final_cb
    CODE:
        int unsafe = (items > 4 && SvTRUE(ST(4)));
        if (!IS_PVCV(worker))
            croak("EV::Future::parallel_map_limit: worker must be a code reference");
        plimit_start(aTHX_ list, worker, limit, final_cb, unsafe);
```

- [ ] **Step 8: Export it**

```perl
our @EXPORT = qw(parallel parallel_limit series race parallel_map parallel_map_limit);
```

- [ ] **Step 9: Run the test to verify it passes**

```bash
make && perl -Mblib -Iblib/arch -Iblib/lib t/04-map.t
```

Expected: PASS, 11 subtests.

- [ ] **Step 10: Run the full suite**

```bash
prove -Iblib/lib -Iblib/arch t/
```

Expected: `All tests successful.`

- [ ] **Step 11: Add POD**

After the `parallel_limit` POD section:

```pod
=head2 parallel_map_limit(\@items, \&worker, $limit, \&final_cb, [$unsafe])

As C<parallel_map>, with at most C<$limit> workers in flight at any time.
C<$limit> is clamped to C<1..scalar(@items)>.

  parallel_map_limit(\@urls, sub {
      my ($url, $done) = @_;
      $client->request($url, sub { ...; $done->() });
  }, 4, sub { print "all fetched\n" });

The item conventions of C<parallel_map> apply unchanged.
```

- [ ] **Step 12: Verify POD**

```bash
podchecker lib/EV/Future.pm
```

Expected: `pod syntax OK.`

- [ ] **Step 13: Commit**

```bash
git add Future.xs lib/EV/Future.pm t/04-map.t
git commit -m "feat: add parallel_map_limit"
```

---

### Task 4: `series_map`

**Files:**
- Modify: `Future.xs` (`series_ctx` at `Future.xs:70-81`, `series_cleanup` at `Future.xs:181`, `_series_next` at `Future.xs:225`, `series_start`, new XSUB)
- Modify: `lib/EV/Future.pm` (`@EXPORT`, POD)
- Modify: `t/04-map.t`

**Interfaces:**
- Consumes: `series_start` from Task 1
- Produces:
  - `series_ctx.worker` (`SV *`)
  - `static void series_start(pTHX_ AV *list, SV *worker, SV *final_cb, int unsafe)`
  - Perl function `series_map(\@items, \&worker, \&final_cb, [$unsafe])`

- [ ] **Step 1: Write the failing test**

Append to `t/04-map.t`, before `done_testing;`:

```perl
subtest 'series_map runs items in order' => sub {
    my @seen;
    my $final = 0;
    series_map([qw(a b c)], sub {
        my ($item, $done) = @_;
        push @seen, $item;
        $done->();
    }, sub { $final++ });

    is_deeply(\@seen, [qw(a b c)], 'items ran in order');
    is($final, 1, 'final_cb fired once');
};

subtest 'series_map async keeps order' => sub {
    our @w;
    my @seen;
    my $final = 0;
    series_map([1, 2, 3], sub {
        my ($item, $done) = @_;
        push @w, EV::timer 0.01, 0, sub { push @seen, $item; $done->() };
    }, sub { $final++; EV::break });

    EV::run;
    is_deeply(\@seen, [1, 2, 3], 'async items still ran in order');
    is($final, 1, 'final_cb fired once');
    @w = ();
};

subtest 'series_map cancellation via truthy done' => sub {
    my @seen;
    my $final = 0;
    series_map([qw(a b c)], sub {
        my ($item, $done) = @_;
        push @seen, $item;
        $done->($item eq 'a' ? 1 : 0);
    }, sub { $final++ });

    is_deeply(\@seen, ['a'], 'truthy done skipped the rest');
    is($final, 1, 'final_cb still fired');
};

subtest 'series_map croaks on a non-coderef worker' => sub {
    eval { series_map([1, 2], [], sub { }) };
    like($@, qr/worker must be a code reference/, 'croaks before dispatch');
};

subtest 'map primitives over a tied array' => sub {
    package TiedList;
    sub TIEARRAY  { bless { d => [@_[1 .. $#_]] }, $_[0] }
    sub FETCH     { $_[0]{d}[$_[1]] }
    sub FETCHSIZE { scalar @{$_[0]{d}} }
    sub STORE     { $_[0]{d}[$_[1]] = $_[2] }
    sub STORESIZE { }

    package main;
    tie my @items, 'TiedList', qw(p q r);

    my @seen;
    my $final = 0;
    series_map(\@items, sub {
        my ($item, $done) = @_;
        push @seen, $item;
        $done->();
    }, sub { $final++ });

    is_deeply(\@seen, [qw(p q r)], 'tied array falls back to av_fetch correctly');
    is($final, 1, 'final_cb fired once');
};

subtest 'map primitive nested inside another primitive' => sub {
    our @w;
    my @seen;
    my $final = 0;

    series([
        sub {
            my $outer = shift;
            parallel_map([qw(a b)], sub {
                my ($item, $done) = @_;
                push @w, EV::timer 0.01, 0, sub { push @seen, $item; $done->() };
            }, sub { $outer->() });
        },
        sub { my $d = shift; push @seen, 'after'; $d->() },
    ], sub { $final++; EV::break });

    EV::run;
    is(scalar @seen, 3, 'inner map and outer series both completed');
    is($seen[-1], 'after', 'outer series waited for the inner map');
    is($final, 1, 'final_cb fired once');
    @w = ();
};

subtest 'unsafe map abandons on an escaping exception' => sub {
    our @w;
    my $final = 0;

    eval {
        parallel_map([1, 2], sub {
            my ($item, $done) = @_;
            if ($item == 1) {
                push @w, EV::timer 0.01, 0, sub { $done->() };
            } else {
                $done->();
                die "boom\n";
            }
        }, sub { $final++ }, 1);
    };
    is($@, "boom\n", 'exception propagated to the caller');

    my $bail = EV::timer 0.2, 0, sub { EV::break };
    EV::run;
    is($final, 0, 'operation abandoned; final_cb never fired');
    @w = ();
};
```

The abandonment subtest pins the 0.06 contract for the map form. The tied-array
subtest exercises the `av_fetch` fallback: `AvREAL(...) && !SvMAGICAL(...)` is
false for a tied array, so the fast `AvARRAY` path is skipped.

- [ ] **Step 2: Run it to verify it fails**

```bash
perl -Mblib -Iblib/arch -Iblib/lib t/04-map.t
```

Expected: FAIL with `Undefined subroutine &main::series_map called`.

- [ ] **Step 3: Add the `worker` field to `series_ctx`**

In the struct at `Future.xs:70-81`, after `SV *final_cb;`:

```c
    SV *worker;    /* NULL for the task form; the worker CV for the map form */
```

- [ ] **Step 4: Release it in `series_cleanup`**

In `series_cleanup` (`Future.xs:181`), after `if (ctx->final_cb) SvREFCNT_dec(ctx->final_cb);`:

```c
    if (ctx->worker) SvREFCNT_dec(ctx->worker);
```

- [ ] **Step 5: Add the `worker` parameter to `series_start`**

Change the signature and forward declaration to:

```c
static void series_start(pTHX_ AV *list, SV *worker, SV *final_cb, int unsafe)
```

initialise `ctx->worker = worker ? SvREFCNT_inc(worker) : NULL;`, and update the `series` XSUB call site to `series_start(aTHX_ list, NULL, final_cb, unsafe);`.

- [ ] **Step 6: Add the dispatch branch in `_series_next`**

In `_series_next` (`Future.xs:225`), first hoist the worker out of the loop.
Immediately after the guard is installed and before `ctx->running = 1;`, add:

```c
    SV *worker = ctx->worker;
    const int have_worker = (worker != NULL);
```

This is not cosmetic. `av_fetch` on a magical (tied) array runs the tie's
`FETCH`, which is arbitrary Perl, and in unsafe mode that Perl can reach an
already-handed-out `$done` and free the context from underneath the loop. Any
`ctx->` dereference between the `*is_freed` check and `call_sv` is therefore a
read-after-free. Task 2 shipped exactly this defect and had to fix it in review;
do not reintroduce it here. `ctx->worker` never changes after construction, so
the hoist is behaviour-preserving.

Then the loop body, which currently reads `if (IS_PVCV(task_sv)) {`. Change the
condition and the argument push:

```c
        if (have_worker || IS_PVCV(task_sv)) {
            if (!ctx->unsafe) {
                if (ctx->current_cv) {
                    CvXSUBANY(ctx->current_cv).any_ptr = NULL;
                    SvREFCNT_dec((SV*)ctx->current_cv);
                }
                CV *cv = newXS(NULL, series_next_cb, __FILE__);
                CvXSUBANY(cv).any_ptr = ctx;
                ctx->current_cv = (CV*)SvREFCNT_inc((SV*)cv);
            } else if (!ctx->current_cv) {
                ctx->current_cv = newXS(NULL, series_next_cb, __FILE__);
                CvXSUBANY(ctx->current_cv).any_ptr = ctx;
            }

            SV *next_rv = NULL;
            dSP;
            ENTER;
            SAVETMPS;
            if (!ctx->unsafe) {
                next_rv = sv_2mortal(newRV_noinc((SV*)ctx->current_cv));
            } else {
                next_rv = sv_2mortal(newRV_inc((SV*)ctx->current_cv));
            }
            PUSHMARK(SP);
            if (have_worker) {
                XPUSHs(task_sv ? sv_2mortal(SvREFCNT_inc(task_sv)) : &PL_sv_undef);
            }
            XPUSHs(next_rv);
            PUTBACK;

            int task_unsafe = ctx->unsafe;
            U32 flags = G_DISCARD | (task_unsafe ? 0 : G_EVAL);
            call_sv(have_worker ? worker : task_sv, flags);
```

As with `_plimit_dispatch`, `_series_next` already dereferences `ctx` inside this
window in code you are not changing. That is pre-existing and tracked
separately; your obligation is only to add no new `ctx->` access between the
`*is_freed` check and `call_sv`.

Leave everything below `call_sv` unchanged, including the `if (!task_unsafe)` error handling, `FREETMPS`, `LEAVE`, and `if (*is_freed) goto done;`.

- [ ] **Step 7: Add the `series_map` XSUB**

Immediately after the `series` XSUB:

```c
void
series_map(list, worker, final_cb, ...)
    AV *list
    SV *worker
    SV *final_cb
    CODE:
        int unsafe = (items > 3 && SvTRUE(ST(3)));
        if (!IS_PVCV(worker))
            croak("EV::Future::series_map: worker must be a code reference");
        series_start(aTHX_ list, worker, final_cb, unsafe);
```

- [ ] **Step 8: Export it**

```perl
our @EXPORT = qw(parallel parallel_limit series race
                 parallel_map parallel_map_limit series_map);
```

- [ ] **Step 9: Run the test to verify it passes**

```bash
make && perl -Mblib -Iblib/arch -Iblib/lib t/04-map.t
```

Expected: PASS, 19 subtests.

- [ ] **Step 10: Run the full suite**

```bash
prove -Iblib/lib -Iblib/arch t/
```

Expected: `All tests successful.`

- [ ] **Step 11: Add POD**

After the `series` POD section:

```pod
=head2 series_map(\@items, \&worker, \&final_cb, [$unsafe])

As C<parallel_map>, but sequential: the worker is called for the next item only
after the previous call has invoked its C<done>. Passing a true value to
C<done> cancels the remaining items, exactly as in C<series>.

  series_map(\@steps, sub {
      my ($step, $done) = @_;
      $etcd->put($step->{key}, $step->{val}, sub { ...; $done->() });
  }, sub { print "all steps applied\n" });

The item conventions of C<parallel_map> apply unchanged.
```

- [ ] **Step 12: Verify POD**

```bash
podchecker lib/EV/Future.pm
```

Expected: `pod syntax OK.`

- [ ] **Step 13: Commit**

```bash
git add Future.xs lib/EV/Future.pm t/04-map.t
git commit -m "feat: add series_map"
```

---

### Task 5: Handle cell and `EV::Future::Handle`

Infrastructure only: allocation, lifetime and `DESTROY`. `cancel`, `pending` and `active` come in Tasks 6 and 7. The deliverable is a handle that can be obtained, can outlive its operation, and can be destroyed in either order without leaking or crashing.

**Files:**
- Modify: `Future.xs` (all four ctx structs, all four cleanup functions, all seven XSUBs, new `MODULE` section)
- Create: `t/05-handle.t`
- Modify: `MANIFEST`, `lib/EV/Future.pm` (POD)

**Interfaces:**
- Consumes: the start functions from Tasks 1 to 4
- Produces:
  - `typedef struct { void *ctx; int kind; int refcnt; } evf_handle;`
  - `#define EVF_KIND_PARALLEL 0`, `EVF_KIND_PLIMIT 1`, `EVF_KIND_SERIES 2`, `EVF_KIND_RACE 3`
  - `static evf_handle *evf_handle_new(pTHX_ void *ctx, int kind)` (refcnt 1, held by the pending Perl object)
  - `static void evf_handle_attach(pTHX_ evf_handle *h, void *ctx)` (refcnt++, for the context's reference)
  - `static void evf_handle_dec(pTHX_ evf_handle *h)`
  - `static SV *evf_handle_wrap(pTHX_ evf_handle *h)` (blesses without incrementing)
  - each ctx gains `evf_handle *h;`
  - each start function gains a trailing `int want_handle` parameter and returns `evf_handle *`
  - Perl package `EV::Future::Handle` with `DESTROY`

- [ ] **Step 1: Write the failing test**

Create `t/05-handle.t`:

```perl
use strict;
use warnings;
use Test::More;
use EV;
use EV::Future;

subtest 'handle is returned in non-void context' => sub {
    our @w;
    my $h = parallel([
        sub { my $d = shift; push @w, EV::timer 0.01, 0, sub { $d->() } },
    ], sub { });

    isa_ok($h, 'EV::Future::Handle');
    EV::run;
    @w = ();
};

subtest 'every primitive returns a handle' => sub {
    our @w;
    my $task = sub { my $d = shift; push @w, EV::timer 0.01, 0, sub { $d->() } };
    my $work = sub { my ($i, $d) = @_; push @w, EV::timer 0.01, 0, sub { $d->() } };

    isa_ok(parallel([$task], sub { }),                    'EV::Future::Handle', 'parallel');
    isa_ok(parallel_limit([$task], 1, sub { }),           'EV::Future::Handle', 'parallel_limit');
    isa_ok(series([$task], sub { }),                      'EV::Future::Handle', 'series');
    isa_ok(race([$task], sub { }),                        'EV::Future::Handle', 'race');
    isa_ok(parallel_map([1], $work, sub { }),             'EV::Future::Handle', 'parallel_map');
    isa_ok(parallel_map_limit([1], $work, 1, sub { }),    'EV::Future::Handle', 'parallel_map_limit');
    isa_ok(series_map([1], $work, sub { }),               'EV::Future::Handle', 'series_map');

    EV::run;
    @w = ();
};

subtest 'handle survives the operation completing' => sub {
    my $h = parallel([sub { shift->() }], sub { });
    isa_ok($h, 'EV::Future::Handle');
    # The operation completed synchronously; the handle must still be a valid
    # object that can be destroyed without crashing.
    undef $h;
    pass('handle destroyed after its operation finished');
};

subtest 'operation survives the handle being destroyed' => sub {
    our @w;
    my $final = 0;
    {
        my $h = parallel([
            sub { my $d = shift; push @w, EV::timer 0.01, 0, sub { $d->() } },
        ], sub { $final++; EV::break });
        isa_ok($h, 'EV::Future::Handle');
    }
    EV::run;
    is($final, 1, 'operation completed after the handle went away');
    @w = ();
};

subtest 'empty task list still yields a handle' => sub {
    my $final = 0;
    my $h = parallel([], sub { $final++ });
    isa_ok($h, 'EV::Future::Handle');
    is($final, 1, 'final_cb fired');
};

done_testing;
```

- [ ] **Step 2: Run it to verify it fails**

```bash
perl -Mblib -Iblib/arch -Iblib/lib t/05-handle.t
```

Expected: FAIL with `The thing isn't a 'EV::Future::Handle'` (the XSUBs currently return nothing).

- [ ] **Step 3: Add the handle type and its helpers**

Insert immediately after the `IS_PVCV` macro at `Future.xs:8`:

```c
#define EVF_KIND_PARALLEL 0
#define EVF_KIND_PLIMIT   1
#define EVF_KIND_SERIES   2
#define EVF_KIND_RACE     3

/* Refcounted cell between a context and its Perl-side handle object. The
   context is freed the moment the operation completes, but the caller may hold
   the handle long after that, so neither side may own the other. Completion
   NULLs `ctx`, which makes every later method call a safe no-op. */
typedef struct {
    void *ctx;    /* NULL once the operation is over */
    int   kind;   /* which primitive, so we can pick the right cleanup */
    int   refcnt; /* held by the Perl object and, while live, by the context */
} evf_handle;

static evf_handle *evf_handle_new(pTHX_ void *ctx, int kind) {
    evf_handle *h;
    Newx(h, 1, evf_handle);
    h->ctx = ctx;
    h->kind = kind;
    h->refcnt = 1; /* the Perl object we are about to hand back */
    return h;
}

static void evf_handle_dec(pTHX_ evf_handle *h) {
    if (!h) return;
    if (--h->refcnt <= 0) Safefree(h);
}

static void evf_handle_attach(pTHX_ evf_handle *h, void *ctx) {
    if (!h) return;
    h->ctx = ctx;
    h->refcnt++; /* the context's reference */
}

static SV *evf_handle_wrap(pTHX_ evf_handle *h) {
    SV *sv = newSViv(PTR2IV(h));
    SV *rv = newRV_noinc(sv);
    sv_bless(rv, gv_stashpv("EV::Future::Handle", GV_ADD));
    SvREADONLY_on(sv);
    return rv; /* does not increment: evf_handle_new already counted this */
}
```

**Refcount contract, stated once because getting it wrong is the main hazard here:** `evf_handle_new` returns a cell with refcnt 1, representing the Perl object the XSUB is about to return. `evf_handle_attach` adds the context's reference, taking it to 2. Cleanup drops the context's reference back to 1. `DESTROY` drops the last one and frees. Because the Perl object's reference is counted *before* dispatch begins, an operation that completes synchronously during dispatch cannot free the cell out from under the XSUB that is about to wrap it.

- [ ] **Step 4: Add the `h` field to all four ctx structs**

Add to `parallel_ctx` (`Future.xs:10`), `series_ctx` (`Future.xs:70`), `plimit_ctx` (`Future.xs:83`) and `race_ctx` (`Future.xs:101`), as the last field before the closing brace:

```c
    evf_handle *h;   /* NULL unless the caller asked for a handle */
```

- [ ] **Step 5: Detach in all four cleanup functions**

In `parallel_cleanup` (`Future.xs:116`), `series_cleanup` (`Future.xs:181`), `plimit_cleanup` (`Future.xs:331`) and `race_cleanup` (`Future.xs:508`), immediately before the closing `Safefree(ctx);`:

```c
    if (ctx->h) {
        ctx->h->ctx = NULL;
        evf_handle_dec(aTHX_ ctx->h);
        ctx->h = NULL;
    }
```

- [ ] **Step 6: Thread `want_handle` through the three start functions**

Change each signature to take a trailing `int want_handle` and return `evf_handle *`:

```c
static evf_handle *parallel_start(pTHX_ AV *list, SV *worker, SV *final_cb, int unsafe, int want_handle);
static evf_handle *series_start(pTHX_ AV *list, SV *worker, SV *final_cb, int unsafe, int want_handle);
static evf_handle *plimit_start(pTHX_ AV *list, SV *worker, I32 limit, SV *final_cb, int unsafe, int want_handle);
```

In each, the empty-list early return becomes (using the matching `EVF_KIND_*`):

```c
    if (len <= 0) {
        if (IS_PVCV(final_cb)) {
            dSP;
            ENTER;
            SAVETMPS;
            PUSHMARK(SP);
            PUTBACK;
            call_sv(final_cb, G_DISCARD | G_VOID);
            FREETMPS;
            LEAVE;
        }
        /* Already over: hand back a dead handle so callers get a uniform
           object rather than undef. */
        return want_handle ? evf_handle_new(aTHX_ NULL, EVF_KIND_PARALLEL) : NULL;
    }
```

Initialise `ctx->h = NULL;` alongside the other context fields, then immediately after the context is fully initialised and **before any dispatch**:

```c
    evf_handle *h = NULL;
    if (want_handle) {
        h = evf_handle_new(aTHX_ NULL, EVF_KIND_PARALLEL);
        evf_handle_attach(aTHX_ h, ctx);
        ctx->h = h;
    }
```

and end each function with `return h;` after the final `LEAVE;`.

**Do not read `ctx` after dispatch returns.** The operation may have completed and freed it. `h` is a local and stays valid.

- [ ] **Step 7: Return the handle from all seven XSUBs**

Each XSUB gains the same two lines. For `parallel`:

```c
void
parallel(list, final_cb, ...)
    AV *list
    SV *final_cb
    CODE:
        int unsafe = (items > 2 && SvTRUE(ST(2)));
        evf_handle *h = parallel_start(aTHX_ list, NULL, final_cb, unsafe,
                                       GIMME_V != G_VOID);
        if (h) {
            ST(0) = sv_2mortal(evf_handle_wrap(aTHX_ h));
            XSRETURN(1);
        }
```

Apply the same pattern to `parallel_map`, `parallel_limit`, `parallel_map_limit`, `series` and `series_map`, passing `GIMME_V != G_VOID` as `want_handle`.

`race` was not extracted into a start function, so wire it inline: add `ctx->h = NULL;` with the other field initialisers, then before the dispatch loop:

```c
        evf_handle *h = NULL;
        if (GIMME_V != G_VOID) {
            h = evf_handle_new(aTHX_ NULL, EVF_KIND_RACE);
            evf_handle_attach(aTHX_ h, ctx);
            ctx->h = h;
        }
```

and after the final `LEAVE;`:

```c
        if (h) {
            ST(0) = sv_2mortal(evf_handle_wrap(aTHX_ h));
            XSRETURN(1);
        }
```

For race's empty-list early return, add before the `return;`:

```c
            if (GIMME_V != G_VOID) {
                ST(0) = sv_2mortal(evf_handle_wrap(aTHX_
                            evf_handle_new(aTHX_ NULL, EVF_KIND_RACE)));
                XSRETURN(1);
            }
```

`ST(0)` is always safe here: every one of these XSUBs takes at least two arguments, so the argument stack has room for one return value.

- [ ] **Step 8: Add the `EV::Future::Handle` MODULE section**

At the very end of `Future.xs`:

```c
MODULE = EV::Future		PACKAGE = EV::Future::Handle

void
DESTROY(self)
    SV *self
    CODE:
        evf_handle *h = INT2PTR(evf_handle *, SvIV(SvRV(self)));
        evf_handle_dec(aTHX_ h);
```

- [ ] **Step 9: Run the test to verify it passes**

```bash
make && perl -Mblib -Iblib/arch -Iblib/lib t/05-handle.t
```

Expected: PASS, 5 subtests.

- [ ] **Step 10: Run the full suite**

```bash
prove -Iblib/lib -Iblib/arch t/
```

Expected: `All tests successful.` All pre-existing tests call the primitives in void context, so none of them should change behaviour.

- [ ] **Step 11: Check for leaks under valgrind**

```bash
valgrind --error-exitcode=99 --errors-for-leak-kinds=definite --leak-check=full \
  --show-leak-kinds=definite "$(perl -MConfig -e 'print $Config{perlpath}')" -Mblib -Iblib/arch -Iblib/lib t/05-handle.t
```

Expected: exit 0. A non-zero exit means the refcount contract in Step 3 was implemented incorrectly; re-read it before changing anything else.

- [ ] **Step 12: Add the test file to MANIFEST**

Insert `t/05-handle.t` after `t/04-map.t`.

- [ ] **Step 13: Commit**

```bash
git add Future.xs t/05-handle.t MANIFEST
git commit -m "feat: return a cancellation handle in non-void context"
```

---

### Task 6: `$h->cancel([$fire])`

**Files:**
- Modify: `Future.xs` (new static function, new XSUB in the `EV::Future::Handle` section)
- Modify: `t/05-handle.t`, `lib/EV/Future.pm` (POD)

**Interfaces:**
- Consumes: `evf_handle`, the four `*_cleanup` functions
- Produces: `static void evf_handle_cancel(pTHX_ evf_handle *h, int fire)`; Perl method `$h->cancel([$fire])`

- [ ] **Step 1: Write the failing test**

Append to `t/05-handle.t`, before `done_testing;`:

```perl
subtest 'cancel stops further dispatch' => sub {
    our @w;
    my ($dispatched, $final) = (0, 0);

    my $h = parallel_map_limit([1 .. 6], sub {
        my ($item, $done) = @_;
        $dispatched++;
        push @w, EV::timer 0.02, 0, sub { $done->() };
    }, 2, sub { $final++ });

    is($dispatched, 2, 'limit respected before cancel');
    $h->cancel;

    my $bail = EV::timer 0.2, 0, sub { EV::break };
    EV::run;

    is($dispatched, 2, 'no further items dispatched after cancel');
    is($final, 0, 'final_cb did not fire');
    @w = ();
};

subtest 'cancel(1) fires final_cb once with no arguments' => sub {
    our @w;
    my ($final, @args) = (0);

    my $h = parallel_limit([
        (sub { my $d = shift; push @w, EV::timer 0.02, 0, sub { $d->() } }) x 4,
    ], 2, sub { $final++; @args = @_ });

    $h->cancel(1);
    is($final, 1, 'final_cb fired exactly once');
    is_deeply(\@args, [], 'final_cb received no arguments');

    my $bail = EV::timer 0.2, 0, sub { EV::break };
    EV::run;
    is($final, 1, 'still exactly once after the loop drained');
    @w = ();
};

subtest 'cancel is idempotent and safe after completion' => sub {
    my $final = 0;
    my $h = parallel([sub { shift->() }], sub { $final++ });
    is($final, 1, 'completed synchronously');

    $h->cancel;
    $h->cancel(1);
    $h->cancel;
    is($final, 1, 'cancel after completion is a no-op');
};

subtest 'cancel from inside a task stops the dispatch loop' => sub {
    our @w;
    my ($dispatched, $final) = (0, 0);
    my $h;

    $h = parallel_map_limit([1 .. 6], sub {
        my ($item, $done) = @_;
        $dispatched++;
        $h->cancel if $item == 1 && $h;
        push @w, EV::timer 0.02, 0, sub { $done->() };
    }, 2, sub { $final++ });

    my $bail = EV::timer 0.2, 0, sub { EV::break };
    EV::run;

    is($final, 0, 'final_cb did not fire');
    cmp_ok($dispatched, '<=', 2, 'dispatch loop stopped');
    @w = ();
};

subtest 'late done() after cancel is ignored' => sub {
    our @w;
    my $final = 0;
    my $saved_done;

    my $h = parallel([
        sub { $saved_done = shift },
        sub { my $d = shift; push @w, EV::timer 0.01, 0, sub { $d->() } },
    ], sub { $final++ });

    $h->cancel;
    $saved_done->();

    my $bail = EV::timer 0.2, 0, sub { EV::break };
    EV::run;
    is($final, 0, 'late done did not resurrect the operation');
    @w = ();
};

subtest 'cancel from inside final_cb is a no-op' => sub {
    our @w;
    my ($final, $err) = (0);
    my $h;

    # The task must be async. $h is assigned only after parallel() returns, so
    # a synchronous task would reach final_cb with $h still undefined and the
    # subtest would never exercise cancel at all.
    $h = parallel([
        sub { my $d = shift; push @w, EV::timer 0.01, 0, sub { $d->() } },
    ], sub {
        $final++;
        eval { $h->cancel(1) };
        $err = $@;
        EV::break;
    });

    EV::run;
    is($final, 1, 'final_cb fired exactly once, not re-entered by cancel(1)');
    ok(!$err, 'cancelling from final_cb did not die') or diag $err;
    @w = ();
};
```

This subtest is load-bearing: by the time `final_cb` runs, cleanup has already
freed the context and set `h->ctx = NULL`, so `cancel` must take its early
return. If it does not, `cancel(1)` either double-frees the context or re-enters
`final_cb`, and the `$final == 1` assertion catches the second case.

- [ ] **Step 2: Run it to verify it fails**

```bash
perl -Mblib -Iblib/arch -Iblib/lib t/05-handle.t
```

Expected: FAIL with `Can't locate object method "cancel" via package "EV::Future::Handle"`.

- [ ] **Step 3: Implement `evf_handle_cancel`**

Add immediately before the `MODULE = EV::Future` line at `Future.xs:785`, so all four cleanup functions are already declared:

```c
/* Cancel can only stop future dispatch; a task already in flight is user code
   sitting in an EV callback and cannot be interrupted. Running the primitive's
   normal cleanup NULLs any_ptr on every outstanding done CV, so those tasks
   calling done later are ignored. Cleanup also sets *(ctx->is_freed_ptr), which
   is how a cancel issued from inside a task makes the dispatch loop bail on its
   next turn. */
static void evf_handle_cancel(pTHX_ evf_handle *h, int fire) {
    SV *cb = NULL;

    if (!h || !h->ctx) return;

    switch (h->kind) {
        case EVF_KIND_PARALLEL: {
            parallel_ctx *ctx = (parallel_ctx *)h->ctx;
            if (fire && IS_PVCV(ctx->final_cb)) cb = sv_2mortal(SvREFCNT_inc(ctx->final_cb));
            parallel_cleanup(aTHX_ &ctx);
            break;
        }
        case EVF_KIND_PLIMIT: {
            plimit_ctx *ctx = (plimit_ctx *)h->ctx;
            if (fire && IS_PVCV(ctx->final_cb)) cb = sv_2mortal(SvREFCNT_inc(ctx->final_cb));
            plimit_cleanup(aTHX_ &ctx);
            break;
        }
        case EVF_KIND_SERIES: {
            series_ctx *ctx = (series_ctx *)h->ctx;
            if (fire && IS_PVCV(ctx->final_cb)) cb = sv_2mortal(SvREFCNT_inc(ctx->final_cb));
            series_cleanup(aTHX_ &ctx);
            break;
        }
        case EVF_KIND_RACE: {
            race_ctx *ctx = (race_ctx *)h->ctx;
            if (fire && IS_PVCV(ctx->final_cb)) cb = sv_2mortal(SvREFCNT_inc(ctx->final_cb));
            race_cleanup(aTHX_ &ctx);
            break;
        }
    }

    if (cb) {
        dSP;
        ENTER;
        SAVETMPS;
        PUSHMARK(SP);
        PUTBACK;
        call_sv(cb, G_DISCARD | G_VOID);
        FREETMPS;
        LEAVE;
    }
}
```

The `sv_2mortal(SvREFCNT_inc(...))` before cleanup is required: cleanup decrements `final_cb`, and without the extra reference the callback could be freed before it is called.

Cleanup sets `h->ctx = NULL` via the detach added in Task 5 Step 5, which is what makes a second `cancel` a no-op. No extra flag is needed.

- [ ] **Step 4: Add the `cancel` XSUB**

In the `EV::Future::Handle` MODULE section, after `DESTROY`:

```c
void
cancel(self, ...)
    SV *self
    CODE:
        evf_handle *h = evf_handle_from_sv(aTHX_ self);
        int fire = (items > 1 && SvTRUE(ST(1)));
        evf_handle_cancel(aTHX_ h, fire);
```

`evf_handle_from_sv` was added in Task 5 and returns NULL for anything that is
not a reference to an integer, so a subclassed or forged handle cannot reach the
`INT2PTR` cast. Use it; do not open-code `INT2PTR(..., SvIV(SvRV(self)))` here.
`evf_handle_cancel` already returns early on a NULL handle.

- [ ] **Step 5: Run the test to verify it passes**

```bash
make && perl -Mblib -Iblib/arch -Iblib/lib t/05-handle.t
```

Expected: PASS, 11 subtests.

- [ ] **Step 6: Run the full suite**

```bash
prove -Iblib/lib -Iblib/arch t/
```

Expected: `All tests successful.`

- [ ] **Step 7: Check for leaks and memory errors**

```bash
valgrind --error-exitcode=99 --errors-for-leak-kinds=definite --leak-check=full \
  --show-leak-kinds=definite "$(perl -MConfig -e 'print $Config{perlpath}')" -Mblib -Iblib/arch -Iblib/lib t/05-handle.t
```

Expected: exit 0.

- [ ] **Step 8: Add POD**

Add a new section after the `race` POD section:

```pod
=head1 CANCELLATION

Called in non-void context, every function returns an C<EV::Future::Handle>.
Called in void context, no handle is allocated and nothing is returned, so
existing call sites are unaffected.

  my $h = parallel_limit(\@tasks, 4, sub { print "done\n" });
  $h->cancel;     # stop dispatching; final_cb never fires
  $h->cancel(1);  # stop dispatching; final_cb fires once, with no arguments

Cancellation stops further dispatch. It cannot interrupt a task already in
flight, because that is your code sitting in an C<EV> callback; such a task
runs to completion, and the C<done> it eventually calls is ignored.

C<cancel> is idempotent and is a no-op once the operation has finished. It may
be called from inside a task, in which case the remaining tasks are not
dispatched, and from C<final_cb>, where it does nothing.

For C<series> this complements the truthy-C<done> form: pass a true value to
C<done> to cancel from inside a task, or use the handle to cancel from outside
one.
```

- [ ] **Step 9: Verify POD**

```bash
podchecker lib/EV/Future.pm
```

Expected: `pod syntax OK.`

- [ ] **Step 10: Commit**

```bash
git add Future.xs t/05-handle.t lib/EV/Future.pm
git commit -m "feat: add handle cancellation"
```

---

### Task 7: `$h->pending` and `$h->active`

**Files:**
- Modify: `Future.xs` (`race_ctx`, race XSUB, new static function, two new XSUBs)
- Modify: `t/05-handle.t`, `lib/EV/Future.pm` (POD)

**Interfaces:**
- Consumes: `evf_handle`, all four ctx structs
- Produces:
  - `race_ctx.total_tasks` (`I32`)
  - `static IV evf_handle_count(pTHX_ evf_handle *h, int want_active)`
  - Perl methods `$h->pending` and `$h->active`

- [ ] **Step 1: Write the failing test**

Append to `t/05-handle.t`, before `done_testing;`:

```perl
subtest 'pending and active on parallel_limit' => sub {
    our @w;
    my @dones;

    my $h = parallel_limit([
        (sub { my $d = shift; push @dones, $d }) x 5,
    ], 2, sub { });

    is($h->active,  2, 'two tasks in flight, matching the limit');
    is($h->pending, 5, 'five tasks not yet completed');
    cmp_ok($h->active, '<', $h->pending, 'queued tasks are pending but not active');

    (shift @dones)->();
    is($h->active,  2, 'a replacement task was dispatched');
    is($h->pending, 4, 'one fewer pending');

    (shift @dones)->() while @dones;
    is($h->pending, 0, 'pending is zero once the operation is over');
    is($h->active,  0, 'active is zero once the operation is over');
    @w = ();
};

subtest 'pending and active on parallel' => sub {
    my @dones;
    my $h = parallel([
        (sub { my $d = shift; push @dones, $d }) x 3,
    ], sub { });

    is($h->pending, 3, 'all three pending');
    is($h->active,  3, 'parallel dispatches everything up front');

    (shift @dones)->();
    is($h->pending, 2, 'pending dropped by one');

    (shift @dones)->() while @dones;
    is($h->pending, 0, 'zero once complete');
};

subtest 'pending and active on series' => sub {
    my @dones;
    my $h = series([
        (sub { my $d = shift; push @dones, $d }) x 3,
    ], sub { });

    is($h->active,  1, 'series always has exactly one task in flight');
    is($h->pending, 3, 'three tasks not yet completed');

    (shift @dones)->();
    is($h->pending, 2, 'pending dropped by one');

    (shift @dones)->() while @dones;
    is($h->pending, 0, 'zero once complete');
    is($h->active,  0, 'zero once complete');
};

subtest 'pending and active on race' => sub {
    my @dones;
    my $h = race([
        (sub { my $d = shift; push @dones, $d }) x 3,
    ], sub { });

    is($h->pending, 3, 'all three dispatched and unsettled');
    is($h->active,  3, 'race collapses active into pending');

    (shift @dones)->('winner');
    is($h->pending, 0, 'settled, so nothing pending');
};

subtest 'counts are zero after cancel' => sub {
    my @dones;
    my $h = parallel([
        (sub { my $d = shift; push @dones, $d }) x 3,
    ], sub { });

    is($h->pending, 3, 'three pending before cancel');
    $h->cancel;
    is($h->pending, 0, 'zero after cancel');
    is($h->active,  0, 'zero after cancel');
};

subtest 'pending is readable from inside a task' => sub {
    our @w;
    my ($seen_pending, $seen_active);
    my $h;

    # The task must be async: $h is assigned only after series() returns, so a
    # synchronous task would see an undefined $h and the test would assert
    # nothing. Deferring through a timer means the handle exists by the time
    # the callback runs.
    $h = series([
        sub {
            my $d = shift;
            push @w, EV::timer 0.01, 0, sub {
                $seen_pending = $h->pending;
                $seen_active  = $h->active;
                $d->();
            };
        },
        sub { shift->() },
    ], sub { EV::break });

    EV::run;
    is($seen_pending, 2, 'pending counted both tasks from inside the first');
    is($seen_active,  1, 'active was 1 from inside a task');
    @w = ();
};

done_testing;
```

- [ ] **Step 2: Run it to verify it fails**

```bash
perl -Mblib -Iblib/arch -Iblib/lib t/05-handle.t
```

Expected: FAIL with `Can't locate object method "active" via package "EV::Future::Handle"`.

- [ ] **Step 3: Add `total_tasks` to `race_ctx`**

In the struct at `Future.xs:101`, after `int settled;`:

```c
    I32 total_tasks;
```

In the `race` XSUB, alongside `ctx->settled = 0;`:

```c
        ctx->total_tasks = len;
```

This is assigned once at construction. There is deliberately no per-task bookkeeping anywhere in this task; every other count comes from a field the dispatch loops already maintain.

- [ ] **Step 4: Implement `evf_handle_count`**

Add immediately after `evf_handle_cancel`:

```c
/* pending: tasks not yet completed, counting both in flight and not yet
   dispatched. active: tasks currently in flight. parallel and race collapse the
   two because they dispatch everything up front and the handle is not returned
   until the XSUB returns. Returns 0 once the operation is over, so
   pending == 0 exactly when it has finished or been cancelled. */
static IV evf_handle_count(pTHX_ evf_handle *h, int want_active) {
    if (!h || !h->ctx) return 0;

    switch (h->kind) {
        case EVF_KIND_PARALLEL: {
            parallel_ctx *ctx = (parallel_ctx *)h->ctx;
            return (IV)ctx->remaining;
        }
        case EVF_KIND_PLIMIT: {
            plimit_ctx *ctx = (plimit_ctx *)h->ctx;
            return want_active ? (IV)ctx->active : (IV)ctx->remaining;
        }
        case EVF_KIND_SERIES: {
            series_ctx *ctx = (series_ctx *)h->ctx;
            IV left;
            if (want_active) return 1;
            left = (IV)ctx->total_tasks - (IV)ctx->current_idx + 1;
            return left < 0 ? 0 : left;
        }
        case EVF_KIND_RACE: {
            race_ctx *ctx = (race_ctx *)h->ctx;
            return (IV)ctx->total_tasks;
        }
    }
    return 0;
}
```

The series clamp matters: truthy-`done` cancellation sets `current_idx = total_tasks`, and `pending` may be read from inside the cancelling task before cleanup runs, which would otherwise yield a misleading count. The formula also reads one too high before the first dispatch, but `current_idx` is incremented before each task is called, so it is already at least 1 by the time any caller can hold the handle.

- [ ] **Step 5: Add the two XSUBs**

In the `EV::Future::Handle` MODULE section, after `cancel`:

```c
IV
pending(self)
    SV *self
    CODE:
        RETVAL = evf_handle_count(aTHX_ evf_handle_from_sv(aTHX_ self), 0);
    OUTPUT:
        RETVAL

IV
active(self)
    SV *self
    CODE:
        RETVAL = evf_handle_count(aTHX_ evf_handle_from_sv(aTHX_ self), 1);
    OUTPUT:
        RETVAL
```

`evf_handle_from_sv` was added in Task 5 and returns NULL for anything that is
not a reference to an integer, so a subclassed or forged handle cannot reach the
`INT2PTR` cast. Use it; do not open-code `INT2PTR(..., SvIV(SvRV(self)))` here.
`evf_handle_count` already returns 0 for a NULL handle.

- [ ] **Step 6: Run the test to verify it passes**

```bash
make && perl -Mblib -Iblib/arch -Iblib/lib t/05-handle.t
```

Expected: PASS, 17 subtests.

- [ ] **Step 7: Run the full suite**

```bash
prove -Iblib/lib -Iblib/arch t/
```

Expected: `All tests successful.`

- [ ] **Step 8: Add POD**

Append to the `=head1 CANCELLATION` section added in Task 6:

```pod
The handle also reports progress. C<pending> counts tasks that have not yet
completed, both in flight and not yet dispatched; C<active> counts those
currently in flight.

  my $h = parallel_limit(\@tasks, 4, sub { });
  printf "%d in flight, %d to go\n", $h->active, $h->pending;

C<parallel> and C<race> report the same value for both, because they dispatch
every task before returning. C<series> always reports one active task while it
is running. Both return 0 once the operation is over, so C<pending == 0> means
it has finished or been cancelled.

In unsafe mode, double-calling C<done> corrupts the completion counter, so
C<pending> under-reports for C<parallel> and C<parallel_limit> for the same
reason C<final_cb> can fire early.
```

- [ ] **Step 9: Verify POD**

```bash
podchecker lib/EV/Future.pm
```

Expected: `pod syntax OK.`

- [ ] **Step 10: Commit**

```bash
git add Future.xs t/05-handle.t lib/EV/Future.pm
git commit -m "feat: add pending and active to the handle"
```

---

### Task 8: Release preparation and full verification

**Files:**
- Modify: `lib/EV/Future.pm` (`$VERSION`), `README.md`, `Changes`

**Interfaces:**
- Consumes: everything from Tasks 1 to 7
- Produces: a release-ready 0.07

- [ ] **Step 1: Bump the version**

In `lib/EV/Future.pm`:

```perl
our $VERSION = '0.07';
```

- [ ] **Step 2: Update the README function list**

Replace the `## Functions` list with:

```markdown
- `parallel(\@tasks, \&final_cb [, $unsafe])` - run all tasks concurrently
- `parallel_limit(\@tasks, $limit, \&final_cb [, $unsafe])` - concurrent with concurrency limit
- `series(\@tasks, \&final_cb [, $unsafe])` - run tasks sequentially
- `race(\@tasks, \&final_cb [, $unsafe])` - first task to finish wins; its args go to `final_cb`
- `parallel_map(\@items, \&worker, \&final_cb [, $unsafe])` - worker per item, concurrently
- `parallel_map_limit(\@items, \&worker, $limit, \&final_cb [, $unsafe])` - worker per item, with a limit
- `series_map(\@items, \&worker, \&final_cb [, $unsafe])` - worker per item, sequentially

Called in non-void context, each returns a handle supporting `cancel`, `pending` and `active`.
```

- [ ] **Step 3: Add the Changes entry**

At the top of `Changes`, below the header line:

Use the date you complete this task, in the same `Day Mon  D YYYY` format the
existing entries use (`date '+%a %b %e %Y'` produces it).

```
0.07    Tue Jul 28 2026
        - Add parallel_map, parallel_map_limit and series_map: a worker is
          called once per item as ($item, $done), replacing the closure-per-item
          idiom. Non-coderef items are data, not instantly-completed tasks
        - All functions now return an EV::Future::Handle in non-void context,
          supporting cancel([$fire]), pending and active. Nothing is allocated
          in void context
        - Behaviour change: assigning from a primitive in scalar context used to
          yield undef and now yields a handle
```

Replace `<date>` with the actual date in the same format the existing entries use.

- [ ] **Step 3a: Exclude the SDD workspace from the dist**

Add `^\\.superpowers/` to `MANIFEST.SKIP`, in the same block as the other tooling-directory patterns. The subagent-driven execution workflow writes briefs and reports to `.superpowers/sdd/`, which `ExtUtils::Manifest` would otherwise sweep into the distribution; `.gitignore` has no bearing on `make dist`.

- [ ] **Step 4: Verify MANIFEST completeness**

```bash
perl -MExtUtils::Manifest=manifind,maniskip -e 'my $s=maniskip(); my $f=manifind(); open my $m,"<","MANIFEST"; my %i = map { (split /\s+/)[0] => 1 } <$m>; chomp %i; my @x = grep { !$s->($_) && !$i{$_} } sort keys %$f; print @x ? "UNLISTED: @x\n" : "MANIFEST clean\n"; my @g = grep { !-e $_ } sort keys %i; print @g ? "MISSING: @g\n" : "all entries exist\n";'
```

Expected: `MANIFEST clean` and `all entries exist`.

- [ ] **Step 5: Run the full suite under ASan and UBSan**

Build a separate sanitizer copy so the normal build is not clobbered:

```bash
SAN=$(mktemp -d)
cp Future.xs Makefile.PL MANIFEST "$SAN"/ && cp -r lib t "$SAN"/
cd "$SAN" && perl Makefile.PL \
  OPTIMIZE='-O1 -g -fsanitize=address,undefined -fno-omit-frame-pointer' \
  LDFLAGS='-fsanitize=address,undefined' && make

export ASAN_OPTIONS='detect_leaks=0:detect_stack_use_after_return=1:abort_on_error=1:halt_on_error=1'
export UBSAN_OPTIONS='print_stacktrace=1:halt_on_error=1'
for t in t/*.t; do
  out=$(LD_PRELOAD=$(cc -print-file-name=libasan.so) perl -Mblib -Iblib/arch -Iblib/lib "$t" 2>&1)
  echo "$out" | grep -qE 'AddressSanitizer|runtime error' && echo "$t: SANITIZER ERROR" || echo "$t: clean"
done
cd -
```

Expected: every file reports `clean`. These are CI's exact flags.

- [ ] **Step 6: Run the full suite under valgrind**

```bash
for t in t/*.t; do
  valgrind --error-exitcode=99 --errors-for-leak-kinds=definite --leak-check=full \
    --show-leak-kinds=definite "$(perl -MConfig -e 'print $Config{perlpath}')" -Mblib -Iblib/arch -Iblib/lib "$t" >/dev/null 2>&1 \
    && echo "$t: clean" || echo "$t: VALGRIND FAILURE"
done
```

Expected: every file reports `clean`.

- [ ] **Step 7: Confirm the benchmark has not regressed**

```bash
perl -Mblib -Iblib/arch -Iblib/lib bench/benchmark.pl
```

Wall-clock on a loaded machine varies by more than 10 percent run to run, so do not read a single run as a regression. If a case looks slow, compare instruction counts against the previous commit instead, which is deterministic:

```bash
valgrind --tool=cachegrind --cache-sim=no --branch-sim=no --cachegrind-out-file=/dev/null \
  "$(perl -MConfig -e 'print $Config{perlpath}')" -Mblib -Iblib/arch -Iblib/lib bench/benchmark.pl 2>&1 | grep 'I refs'
```

Expected: within roughly 1 percent of the pre-change count. Nothing in this plan adds work to a dispatch loop, so a real regression means a `worker` branch was placed inside a hot path rather than beside the existing `IS_PVCV` test.

- [ ] **Step 8: Commit**

```bash
git add lib/EV/Future.pm README.md Changes
git commit -m "release: 0.07 with map primitives and cancellation handles"
```

---

## Self-Review Notes

Checked against the spec:

- Map primitives, worker signature, `worker` field approach, non-coderef items as data, croaking worker, `sv_2mortal(SvREFCNT_inc())` item passing: Tasks 2 to 4.
- `GIMME_V` allocation, refcounted cell, `DESTROY`: Task 5.
- `cancel([$fire])` semantics including re-entrancy and idempotence: Task 6.
- `pending`/`active` table, `race_ctx.total_tasks`, series clamp, `pending == 0` invariant, unsafe double-call caveat: Task 7.
- Behaviour change note, README, Changes, MANIFEST: Tasks 2, 5 and 8.
- Testing section of the spec: covered across `t/04-map.t` and `t/05-handle.t`, with ASan and valgrind gates in Task 8 and an early valgrind check in Tasks 5 and 6 where the refcount work lands.

Two spec items are deliberately not implemented as separate tasks: `race_map` and the error/result channel are both listed as out of scope.

One spec test requirement has no Perl-level test, deliberately. "Void context
allocates nothing" is not observable from Perl: there is no handle to inspect,
which is the entire point. It is covered instead by Task 8 Step 7, where the
cachegrind instruction count for `bench/benchmark.pl` (which calls every
primitive in void context) must land within roughly 1 percent of the
pre-change count. If `GIMME_V` were wired up wrongly and a handle were built on
every call, that count would rise measurably. Do not add a Perl test that
pretends to check this.
