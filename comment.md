Plan for #72, built from my own reproduction (comment above: https://github.com/codepath/pathreview-ai301-fa26-s3/issues/72#issuecomment-5865096659). 

**Observed:** `verify_password("test-password", "not-a-valid-password-hash")` raises `passlib.exc.UnknownHashError` instead of returning `False`. The H-05 test is `xfail(strict=True)`.

**Diagnosis:** `verify_password()` in `core/security.py` calls `pwd_context.verify(...)` with no exception handling, so passlib's `UnknownHashError` from identifying the hash format reaches the caller. This comes from my repro plus reading the code; I will re-confirm it with a `--runxfail` run before changing anything.

**Change (two files, about 15 lines):**
- `core/security.py`: catch `UnknownHashError` in `verify_password()` and return `False`.
- `tests/unit/test_security.py`: remove the `xfail` marker on `test_verify_with_wrong_hash_format` (strict xfail would fail CI once fixed) and add one test with the input from my repro.

**Not touching:** `hash_password`, the JWT functions, the `CryptContext` config, and other exception types. A corrupt hash that looks like bcrypt may raise `ValueError` instead; I have not checked that and will leave it out unless a maintainer wants it covered.

**Test plan:** re-run the repro and the H-05 test; both should return `False` after the fix. `pytest tests/unit/test_security.py` should pass in full so valid hashes still verify, and `make lint` and `make typecheck` should pass.

Open question for review: returning `False` silently makes a corrupt stored hash look like a wrong password. I am not adding logging; say if you want it.
