# Known issues carried past 0.06

Found during the 0.06 review cycle, triaged as non-blocking for that release.
None is a regression introduced by 0.06 unless stated. This file is excluded
from the distribution by `MANIFEST.SKIP`.

## Checked and closed: the CI valgrind gate is genuine

Recorded so it is not re-raised. Valgrind does not follow an `execve` unless
`--trace-children=yes` is passed. Where `perl` on `PATH` is a wrapper script
that execs the real interpreter, `valgrind ... perl t/foo.t` therefore
instruments nothing: it prints no `ERROR SUMMARY` and exits with the test
suite's own status, which is indistinguishable from a clean run. Every local
valgrind claim made during the 0.06 cycle was vacuous for this reason until it
was caught, and the fix is to invoke `$Config{perlpath}` and to confirm an
`ERROR SUMMARY` line is actually present rather than trusting the exit code.

**This never affected CI.** `shogo82148/actions-setup-perl` puts the real
interpreter on `PATH`, not a wrapper, so the workflow's valgrind job has always
instrumented the real process. The workflow needs no change.

Re-check it on any run rather than trusting this note; the job log should carry
one `ERROR SUMMARY` line per test file, each with `definitely lost: 0 bytes`:

    gh run view <run-id> --log --job \
      "$(gh run view <run-id> --json jobs \
         --jq '.jobs[] | select(.name=="Valgrind") | .databaseId')" \
      | grep -cE 'ERROR SUMMARY'

An earlier version of this file cited a specific run ID as the evidence. That
run was later deleted in a routine cleanup and the citation rotted within a day.
Cite the check, not the artifact.

The hazard is local toolchains: plenv, plus anything else that shims `perl`.
Anyone reproducing the sanitizer results by hand should use `$Config{perlpath}`
and check for the summary line. ASan is unaffected either way, because
`LD_PRELOAD` and compile-time instrumentation both survive `exec`.

## Checked and closed: `pending` really does read 1 after a truthy `done`

The CANCELLATION POD says a task that cancels itself with a truthy `done` and
then reads the handle sees `pending == 1` until it returns, whereas
`$h->cancel` from the same position gives 0. That has now been challenged once
and measured three times; it is correct. Recorded so it is not challenged a
fourth time.

The trap is the observation point. Read the counters from `final_cb` and both
forms show 0, because `series_cleanup` has already detached the handle by then.
The documented claim is specifically about reading from *inside* the cancelling
task, before it returns, and it must be a task dispatched from the event loop:
a synchronously dispatched first task runs before the primitive has returned, so
the handle variable is not assigned yet and there is nothing to read.

    # inside the cancelling task, task dispatched from the event loop
    before done:    pending=2 active=1
    after done(1):  pending=1 active=1     <- the documented contrast
    after cancel:   pending=0 active=0

## 1. Tied and magical arrays are silently skipped by the task form

`parallel`, `series`, `parallel_limit` and `race` skip every element of a tied
or otherwise magical array and fire `final_cb` immediately, as though the list
were all-undef. `IS_PVCV` (`Future.xs:8`) tests `SvROK` without get magic, so it
sees the un-fetched `SVt_PVLV` that `av_fetch` returns for a tied element and
treats it as a non-coderef.

Present since at least 0.05; confirmed by building `d3527cc`. The map form is
unaffected, because the worker reads the element it is handed and that performs
the fetch. `t/01-basic.t` has two magical-array subtests but both assert only
`ok(!$@)`, so they pass vacuously.

Documented as a limitation in 0.06. The fix is `SvGETMAGIC(task_sv)` before
`IS_PVCV`, and it is not free: it runs arbitrary Perl at exactly the point
between `av_fetch` and the context dereferences in all four dispatch sites, so
it must be paired with hoisting those dereferences or re-checking `*is_freed`
after the fetch. Fix and hazard are the same change.

Worth recording, because two review rounds went the other way before this was
settled: `av_fetch(av, key >= 0, lval = 0)` does **not** run a tie's `FETCH`. It
returns a lazy `SVt_PVLV` carrying `PERL_MAGIC_tiedelem`, and `FETCH` runs later
when something reads that SV with get magic. There is therefore no re-entrancy
window between `av_fetch` and the context dereferences today. Fixing this bug
creates one.

## 2. Checked and closed: handle payload moved to `PERL_MAGIC_ext`

Resolved. The C pointer to `evf_handle` was previously stored in the IV slot of
the blessed scalar referent. That allowed forged handles (`bless \(my $x = 12345),
'EV::Future::Handle'`) to segfault on access or destruction, and `Clone::clone($h)`
to duplicate the pointer and trigger a double-free on destruction.

The handle now attaches the `evf_handle` pointer via `PERL_MAGIC_ext` with a
dedicated `MGVTBL` (`&evf_handle_vtbl`). `evf_handle_from_sv` uses `mg_findext`:
- Forged handles carry no magic; `mg_findext` returns NULL, and methods no-op
  safely without segfaulting.
- `Storable` drops the magic, and `Clone::clone` copies it without our vtable,
  so `mg_findext` finds nothing on either copy: it is an inert dead handle that
  neither duplicates the C cell nor double-frees.
- Explicit `DESTROY` clears `mg->mg_ptr` to prevent any double-free if `DESTROY`
  is called multiple times.
Verified under Valgrind: 0 errors.

## 3. Checked and closed: `perldoc EV::Future::Handle` resolves

Resolved. Added `lib/EV/Future/Handle.pod` documenting the `EV::Future::Handle`
class and methods, registered it in `MANIFEST`, and listed it in `Makefile.PL`'s
`PM` so it is installed with a man page. `perldoc EV::Future::Handle` resolves
for an installed copy too.

## 4. Unsafe-mode double-call can push `parallel_limit` past its own limit

Double-calling `done` in unsafe mode corrupts the completion counter; that is
documented. What is not documented is that it also drives `ctx->active`
negative, which loosens the `ctx->active < ctx->limit` dispatch predicate and
lets `parallel_limit` exceed its stated concurrency bound.

`active()` is clamped at 0 as of 0.06, but only where it crosses into Perl; the
dispatch predicate still sees the true negative value, deliberately, so the
clamp cannot mask a scheduling problem. Pre-existing.

## 5. Checked and closed: `$limit` widening clamped in NV space

Resolved. `parallel_limit` and `parallel_map_limit` now take `$limit` as an `SV *`
and clamp in `NV` space (`SvNV`) against `1.0` and `(NV)len` before narrowing to `IV`.
This prevents signed truncation or negative wrapping on 32-bit IV perls for values
exceeding `IV_MAX` (e.g. `2**31`, `2**35`, or floating-point values like `1e12`).

## 6. Smaller items

- [CLOSED] `series_cleanup` decrements `current_cv` without NULLing it: hardened by
  clearing `ctx->current_cv = NULL` before `SvREFCNT_dec`.
- [CLOSED] `race_task_done` runs `race_cleanup` before copying the `done`
  arguments: a tied argument whose FETCH cancels the race used to run the
  cleanup a second time. The arguments are pinned first, because the cleanup
  frees the losing tasks and a loser's DESTROY can free one of them.
- [CLOSED] `race`'s void empty-list path uses `XSRETURN_EMPTY;` instead of bare `return;`.
- [CLOSED] The `active` clamp in unsafe mode is now tested in `t/05-handle.t`.
- [CLOSED] `Makefile.PL` now validates that `MM->parse_version` returned a defined, non-`undef` version.
