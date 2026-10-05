# Plan: issue #72, `verify_password` raises `UnknownHashError` on malformed stored hashes

Issue: https://github.com/codepath/pathreview-ai301-fa26-s3/issues/72
Branch (to create at build time): `fix/72-verify-password-malformed-hash`

## Diagnosis

**Cause:** `verify_password()` in `core/security.py` calls
`pwd_context.verify(plain_password, hashed_password)` directly, with no
exception handling. When the stored hash is not a recognizable format,
passlib raises `UnknownHashError` while identifying the scheme, and
nothing in `verify_password()` catches it, so it propagates to the
caller instead of returning `False`. This is the behavior the issue
title names (`verify_password` raises `UnknownHashError` on malformed
stored hashes instead of returning False).

**Evidence from my repro comment on #72** (posted in unit 2,
https://github.com/codepath/pathreview-ai301-fa26-s3/issues/72#issuecomment-5865096659):

> The verification call raises: `passlib.exc.UnknownHashError` instead of returning `False`.

> The failure occurs in the password verification path when Passlib attempts to identify the stored hash format. `UnknownHashError` is not currently handled by `verify_password()`, so the exception propagates to the caller.

> The existing `H-05` test is marked `xfail`, which is consistent with the current implementation not satisfying the expected fail-closed behavior.

The repro used this case:

```python
def test_verify_password_malformed_hash():
    assert verify_password("test-password", "not-a-valid-password-hash") is False
```

**Read against the code** (`core/security.py`, current `main`):

```python
def verify_password(plain_password: str, hashed_password: str) -> bool:
    return bool(pwd_context.verify(plain_password, hashed_password))
```

There is no `try`/`except` in this function, which matches the repro's
observation that the exception escapes. By contrast, `decode_access_token()`
in the same file already catches `JWTError` and returns `None`, so
failing closed on bad input is an established pattern in this module.

**What I observed vs. what I infer.** Observed (repro comment): the call
raises `UnknownHashError`, and the H-05 test is marked `xfail`. Inferred
(from reading the code above): the missing exception handling is the
cause. I have not yet re-run this on my current machine; my local
environment does not have the project dependencies installed yet, so I
will confirm the failing behavior again at the start of the build.

## Scope

**In scope (one bounded change):**

- Make `verify_password()` return `False` when passlib raises
  `UnknownHashError`, so a malformed or unrecognized stored hash fails
  closed. Reason: the issue asks for fail-closed verification, and a
  raised exception forces every caller to handle a malformed hash
  separately (or crash).
- Remove the `@pytest.mark.xfail(strict=True, ...)` marker from the H-05
  test. Reason: the marker is `strict=True`, so once the fix makes the
  test pass, CI fails with `XPASS(strict)` until the marker is removed
  (`docs/CONTRIBUTING.md`, "Working on a seeded bug"). The issue also
  asks for it to be removed.
- Add one regression test using the exact input from my repro.

**Not in scope (will not touch):**

- Hashing (`hash_password`) and JWT functions (`create_access_token`,
  `decode_access_token`).
- The bcrypt `CryptContext` configuration (`schemes`, `deprecated`).
- Other exception types (for example a `ValueError` for a hash that
  looks like bcrypt but is corrupt). The issue names only
  `UnknownHashError`; see Risks.
- Callers: a search of the repo finds no use of `verify_password`
  outside `core/security.py` and `tests/unit/test_security.py`, so there
  is no caller code to change.
- Logging or alerting when a malformed hash is seen.
- Any other seeded bug, and the `pyproject.toml` suppressions (there is
  no `core.security` entry there to remove).

## Files I will touch

| File | Change | Why |
|---|---|---|
| `core/security.py` | Import `UnknownHashError` from `passlib.exc`; wrap the `verify` call in `try`/`except UnknownHashError: return False`; note the behavior in the docstring. About 5 lines. | This is where the exception escapes. |
| `tests/unit/test_security.py` | Delete the `xfail` decorator on `test_verify_with_wrong_hash_format`; add one test that checks the repro's input. About 10 lines. | Strict xfail would fail CI once fixed; the new test covers the repro's exact input. |

No files are added or deleted. Total change is well under 500 lines
(roughly 15).

## Approach

1. Create branch `fix/72-verify-password-malformed-hash` on my fork.
2. Install dependencies and re-run the H-05 test with `--runxfail` to
   confirm `UnknownHashError` still occurs on current `main` (the
   "before" output).
3. In `core/security.py`, add `from passlib.exc import UnknownHashError`
   and change `verify_password` to:

   ```python
   try:
       return bool(pwd_context.verify(plain_password, hashed_password))
   except UnknownHashError:
       return False
   ```

   and update the docstring "Returns" to say it returns `False` for an
   unrecognized stored hash.
4. In `tests/unit/test_security.py`, remove the
   `@pytest.mark.xfail(...)` block above `test_verify_with_wrong_hash_format`.
5. In the same file, add `test_verify_password_malformed_hash` (the case
   from my repro comment: `"not-a-valid-password-hash"`).
6. Run the checks listed in the test plan, then read the full diff
   before committing. `plan.md` and `comment.md` stay out of the commit.

## Test plan

**Reproduction steps re-run** (from my unit 2 repro; the "before" is the
output I posted):

1. From the repo root with dependencies installed, run the existing
   H-05 test with the marker ignored:
   `pytest tests/unit/test_security.py::TestSecurity::test_verify_with_wrong_hash_format --runxfail`
2. Run the repro's case:
   `verify_password("test-password", "not-a-valid-password-hash")`
   (as the new test).

**Before the fix:** both raise `passlib.exc.UnknownHashError`; with the
marker in place the H-05 test reports `xfailed`.

**After the fix, I expect:**

- `verify_password("password", "not_a_valid_bcrypt_hash")` returns
  `False` (H-05 test passes with no `xfail` marker, shown as `passed`,
  not `xfailed` or `XPASS`).
- `verify_password("test-password", "not-a-valid-password-hash")`
  returns `False` (new test passes).

**Regression checks (nothing else changed):**

- `pytest tests/unit/test_security.py` passes in full, including
  `test_verify_password_correct` and `test_verify_password_incorrect`,
  so valid hashes still return `True` and wrong passwords still return
  `False`.
- `make lint` and `make typecheck` pass.
- On the unfixed code, the new test fails (raises `UnknownHashError`),
  so it can detect the bug.

## Risks and unknowns

- **Not yet re-run locally.** My dependencies are not installed on this
  machine. My claim that the exception comes from missing handling rests
  on my posted repro and on reading the code, and I will re-confirm it
  with the "before" run in step 2 before changing code.
- **Other malformed-hash errors.** A string that starts like a bcrypt
  hash but is corrupt may raise a different exception (such as
  `ValueError`) rather than `UnknownHashError`. I have not checked this.
  I am leaving it out of scope because the issue names only
  `UnknownHashError`, and will say so in the PR rather than widening the
  `except` without evidence.
- **Failing closed hides the problem.** Returning `False` silently
  means a corrupt stored hash looks like a wrong password. I am not
  adding logging, because it is not asked for and the repo's current
  pattern (`decode_access_token`) does not log either. A maintainer may
  prefer logging; I will raise it in the PR.
- **Shared issue.** Other students have claimed and reproduced #72. My
  plan is built from my own repro; if a maintainer states a different
  preferred approach in the thread, I will follow it and record the
  change under Deviations.

## Deviations

<!-- Fill in after the build: what changed from this plan and why. If
nothing changed, say so in my own words. -->

The code change matches the plan: same two files, same `UnknownHashError`
catch, `xfail` marker removed, one new test using the repro's input. The
"before" run with `--runxfail` confirmed `UnknownHashError` on unfixed code,
and the new test also fails without the fix. `pytest tests/unit/test_security.py`
(26 passed), `make lint` and `make typecheck` pass.

One difference, in the plan's step 1: the branch is `fix/72-hash-error`
(the branch already existed as `fix/issue-72-hash-error` and I renamed it to
fit the `fix/<issue-number>-<slug>` house rule), not
`fix/72-verify-password-malformed-hash`. It only affects the branch name, so
the posted plan comment (which never names the branch) is still true and
needs no update.

The Risks section said I had not checked whether a corrupt bcrypt-looking
hash raises something other than `UnknownHashError`. I have now: on the fixed
code, `verify_password("password", "$2b$12$abc")` still raises
`ValueError: salt too small (bcrypt requires exactly 22 chars)`, while
`"not-a-valid-password-hash"` and `""` return `False`. Two other students'
repro reports on #72 show the same result. I am still leaving it out of
scope, since the issue names only `UnknownHashError`, and will raise it in
the PR so a maintainer can decide whether to widen the `except`. The posted
plan comment already says this case may raise `ValueError` and is out of
scope, so it is still true and needs no update.

Two other small notes: the docstring "Returns" line says it returns `False`
for an unrecognized stored hash, as planned, and the dependencies were
installed with a local `.venv` (pip install, no DB/seed/npm steps).