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

## 2. The handle's payload is an unvalidated pointer: two consequences

`EV::Future::Handle` is a blessed reference to an integer holding a C pointer to
one refcounted cell. `evf_handle_from_sv` accepts any `SvROK` + `SvIOK` invocant
and dereferences whatever integer it finds. That single design choice produces
two distinct problems, and one change fixes both.

**a. `Clone::clone` on a handle is a double free.** Two duplication routes are
defended: thread cloning via `CLONE_SKIP`, and `Storable` via
`STORABLE_freeze`/`STORABLE_thaw`, which yield an inert dead handle. A deep
cloner that copies the underlying scalar directly bypasses both;
`Clone::clone($h)` produces a second live owner and aborts with `double free or
corruption`.

**b. A forged handle segfaults.** `bless \(my $x = 12345), 'EV::Future::Handle'`
then calling `cancel`, `pending`, `active` or letting it be destroyed
dereferences address 12345. Reproduced: SIGSEGV, exit 139. Not reachable through
the module's own API, since real payloads are `SvREADONLY`, `STORABLE_thaw`
zeroes to 0 which the accessor tolerates, and `CLONE_SKIP` yields undef; it
needs deliberate forgery, which is the standard blessed-pointer caveat for any
XS class. A hashref-based subclass is rejected safely (it is not `SvIOK`); only
an integer-backed one gets through.

Both are new in 0.06, since the handle is new. Both are closed by moving the
pointer out of the IV slot and into `PERL_MAGIC_ext` magic attached with
`sv_magicext`: a forged or foreign invocant then carries no magic, `mg_findext`
returns NULL, and the accessor no-ops. That is a representation change rather
than a patch, which is why it did not land in 0.06. Whether it also closes (a)
depends on whether the cloner copies magic; verify that rather than assuming it.

## 3. `perldoc EV::Future::Handle` does not resolve

The package is declared in `lib/EV/Future.pm` and documented there, and
`provides` metadata points PAUSE and MetaCPAN at it correctly. But
`Pod::Perldoc` resolves a module name to a file path in `@INC` and has no
fallback into a parent module's POD, so no work inside `lib/EV/Future.pm` can
fix this.

A POD-only `lib/EV/Future/Handle.pod` would fix `perldoc` with a `MANIFEST`
entry alone, with no `require` and no load-order change. A real `.pm` is needed
only for one further case: `Storable::retrieve` of a stored handle in a process
that has not loaded `EV::Future` now dies with `Can't locate
EV/Future/Handle.pm`, because Storable's hook path requires the blessed package.
That is strictly safer than the behaviour before the `STORABLE_*` hooks were
added, which silently produced a blessed stale cross-process pointer whose
`DESTROY` would free a wild address. (Both states are internal to the 0.06
cycle; there is no released version with the handle but without the hooks.)

## 4. Unsafe-mode double-call can push `parallel_limit` past its own limit

Double-calling `done` in unsafe mode corrupts the completion counter; that is
documented. What is not documented is that it also drives `ctx->active`
negative, which loosens the `ctx->active < ctx->limit` dispatch predicate and
lets `parallel_limit` exceed its stated concurrency bound.

`active()` is clamped at 0 as of 0.06, but only where it crosses into Perl; the
dispatch predicate still sees the true negative value, deliberately, so the
clamp cannot mask a scheduling problem. Pre-existing.

## 5. The `$limit` widening is still bounded by IV width

0.06 changed `parallel_limit`'s and `parallel_map_limit`'s `$limit` from `I32`
to `IV` because the old parameter truncated: `limit => 2**31` silently became 1
and ran the whole list sequentially. On a 64-bit-`IV` perl that is fully fixed,
verified at 2**31, 2**32 and 2**60.

On a perl built with a 32-bit `IV` the same class of truncation remains for
values above `IV_MAX`, and a value that wraps negative is clamped back to 1,
which is the original symptom. Nothing in CI builds such a perl, so this is
untested rather than known-broken. A width-independent fix reads the argument as
an `SV *` and clamps in `NV` space before narrowing.

## 6. Smaller items

- `series_cleanup` decrements `current_cv` without NULLing it. Unreachable, for
  the same reason the `cvs`/`num_cvs` NULLing added in 0.06 is unreachable, but
  it is now the only cleanup without that hardening.
- The three scripts in `eg/` all use the closure-per-item idiom that
  `parallel_map` was added to replace, so the distribution's most visible
  examples do not demonstrate its headline feature.
- `race`'s void empty-list path uses a bare `return;` where every other exit
  goes through `XSRETURN`. Harmless and pre-existing.
- The `active` clamp has no test.
- `MM->parse_version` returns the string `"undef"` rather than failing if it
  cannot parse a version, which would put `"undef"` into `provides`. Not
  reachable today, since `VERSION_FROM` would fail loudly first.
