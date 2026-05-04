---
name: python-2-to-3
description: Migrate a Python 2 codebase to Python 3.12+ — `2to3` baseline, the parts the tool misses (bytes/str semantics, dict ordering, divisions, exception syntax), encoding handling, and the dependency upgrades. Use only when stuck on legacy Py2 — Python 2 has been EOL since 2020 but lives on in regulated/government/embedded.
---

# Migrate Python 2 → 3

Python 2 has been EOL since January 2020. If you still have Py2 code, it's because legacy environment locks (regulated, government, embedded) prevent upgrade — or because the code base is large enough that nobody wanted to bite the bullet. This skill is the bullet.

## When to use

- Stuck on Python 2.7 and finally given approval to upgrade.
- Forking a Py2-only library to make it 3-compatible.
- Dual-stack (2 and 3) codebase being collapsed to 3-only.

## When NOT to use

- New code — should be Python 3 from day 1.
- Already on Python 3.x — see `python-version-upgrade` (different skill, not in this repo yet).
- Using libraries that are *truly* Py2-only — fork or replace, not convert.

## The honest cost

A clean ~10kLOC Py2 codebase with decent tests: 1–4 weeks for one engineer.
A messy ~100kLOC codebase, no tests, custom encoding handling: 3–9 months.

The hardest parts: **bytes vs str**, **dict ordering** (when code accidentally depends on Py3.7+ insertion order *or* on Py2's hash randomization), and **division semantics**.

## Step 0: prerequisites

- Python 3.12+ installed locally (3.12+ recommended; covers Py3.10+ improvements like better error messages).
- Type-checking helper: `mypy` or `pyright` (not strictly needed but catches issues).
- Tests must run on Py2 *first*. If they don't, fix that before migrating. Tests are your safety net.

## Step 1: run `2to3` (the baseline)

```bash
2to3 -w -n src/                        # in-place; -n skips backups (you have git)
```

`2to3` handles the mechanical changes:

| Change | Before | After |
|---|---|---|
| Print | `print "hi"` | `print("hi")` |
| Exception syntax | `except E, e:` | `except E as e:` |
| `xrange` | `xrange(10)` | `range(10)` |
| `iteritems()` | `d.iteritems()` | `d.items()` |
| `unicode` | `unicode("x")` | `str("x")` |
| `basestring` | `isinstance(x, basestring)` | `isinstance(x, str)` |
| Octal | `0755` | `0o755` |
| Imports | `import urllib2` | `import urllib.request` |

After this pass, your code *might* run on Py3. Tests will tell you what `2to3` missed.

## Step 2: bytes vs str (the hardest part)

Python 2: `str` was bytes. `unicode` was text. They auto-converted in surprising ways.
Python 3: `bytes` is bytes. `str` is text (always Unicode). They DO NOT auto-convert.

```python
# Py2 (works)
x = "hello"
y = u"héllo"
z = x + y                              # auto-decodes x as ASCII; works for ASCII

# Py3 — needs explicit handling
x = b"hello"                           # bytes
y = "héllo"                            # str (Unicode)
z = x + y                              # TypeError
z = x.decode("utf-8") + y              # ok
```

Where bytes/str confusion bites:
- File I/O: `open(path)` opens text mode by default in Py3 (not bytes). Use `open(path, "rb")` for binary.
- Network: socket reads are bytes; HTTP libs return bytes; you `.decode()` at the boundary.
- Regex: `re` operates on str OR bytes; mixing throws.
- Subprocess: `subprocess.check_output()` returns bytes by default; pass `text=True` for str.
- Hashing/crypto: hashes operate on bytes. `hashlib.sha256(b"x").digest()` — note the `b`.

**Rule:** decode at the boundary (input), encode at the boundary (output). Internal code is all str.

```python
# Reading user input from a file
with open("input.txt", "rb") as f:
    raw = f.read()                      # bytes
text = raw.decode("utf-8")              # convert at boundary

# Writing to network
payload = json.dumps({"x": 1}).encode("utf-8")
sock.send(payload)
```

## Step 3: division

Python 2: `5 / 2 = 2` (integer division when both operands are int).
Python 3: `5 / 2 = 2.5` (always float division).

To get integer division in Py3: `5 // 2 = 2`.

`2to3` does NOT change divisions for you (it can't always tell intent). Audit divisions manually:

```bash
grep -n " / " src/                     # find all divisions
```

For each: is the result expected to be an int (use `//`) or a float (leave `/`)?

Common bug source: pagination, batch sizes, percentages.

## Step 4: dict ordering

Python 3.7+: dicts preserve insertion order officially. Code now *can* depend on this.

But:
- Your old Py2 code might depend on hash-randomization order (effectively random per run). That randomness used to mask race conditions, etc. Now order is deterministic; old random-passing tests reveal real bugs.
- Tests that asserted on dict iteration order: usually now correct; sometimes need updating because the "right" order is now the insertion order, not the alphabetical one some test fixture assumed.

## Step 5: integer types

Py2: `int` and `long` were separate. `1L` was a literal long.
Py3: just `int`. No `long`. `int` is arbitrary precision.

`2to3` strips the `L` suffix. But:
- `isinstance(x, long)` becomes... what? `2to3` converts to `int`. Verify.
- Code that checked types explicitly often had the wrong intent; now's a chance to clean up.

## Step 6: exception types and chaining

Py2: `except E, e:`
Py3: `except E as e:`

Already handled by `2to3`. But Py3 has chained exceptions:

```python
try:
    do_thing()
except SomeError as e:
    raise NewError("context") from e        # explicit chaining
```

Py3 *implicitly* chains too: if a `try`/`except` block raises a new exception, the original is preserved. The traceback shows both. Migrate `raise SomeError("msg, original=" + str(e))` patterns to `raise from`.

## Step 7: dependency upgrades

Most Py2-only libraries have Py3 versions. The migration shopping list:

| Py2-only | Py3 equivalent |
|---|---|
| `urllib2` | `urllib.request` |
| `httplib` | `http.client` (or `requests`/`httpx`) |
| `Queue` | `queue` |
| `cPickle` | `pickle` |
| `StringIO.StringIO` | `io.StringIO` (text) or `io.BytesIO` (bytes) |
| `unittest2` | stdlib `unittest` |
| `mock` | `unittest.mock` (stdlib in 3.3+) |
| `enum34` | stdlib `enum` |

For libraries that are genuinely Py2-only (rare; usually unmaintained):
1. Check if the library has a Py3 fork on GitHub.
2. Check if a successor exists with a different name.
3. Worst case: vendor the library and port it yourself (often easier than rewriting around it).

## Step 8: print → logging

`2to3` converts `print` to `print()` calls. But many "prints" are really logs. Replace with `logging`:

```python
# Before
print("user signed up")

# After
import logging
logger = logging.getLogger(__name__)
logger.info("user signed up", extra={"user_id": user.id})
```

This is bigger than just Py3 migration — it's an observability cleanup. Worth doing during the migration since you're touching the code anyway.

## Step 9: encoding declarations

Py2 source files often had `# -*- coding: utf-8 -*-` at the top. Py3 defaults to UTF-8. Remove the line; it does nothing.

## Step 10: tests, tests, tests

After all the above, run tests. Then run them again under different conditions:

- Different locales (`LC_ALL=C` vs `LC_ALL=en_US.UTF-8`).
- Different Python minor versions (3.10 / 3.11 / 3.12).
- Production-like data (not just unit-test fixtures).

Bytes/str bugs hide in the data. Test fixtures with only ASCII won't catch them.

## Anti-patterns

- **Migrating without tests** — you have no safety net. Write smoke tests first if no real tests exist.
- **One big-bang PR** — module-by-module migration is much safer. Use `six` (Py2/3 compat lib) for the dual-stack interim.
- **Suppressing `DeprecationWarning`** during migration — they're guiding you. Read them.
- **Trusting `2to3` blindly** — it makes mechanical changes. Logic verification is on you.
- **Mixing bytes and str** in internal APIs — pick one and convert at boundaries.
- **Keeping `from __future__ import unicode_literals` in Py3 code** — was a Py2 backport; meaningless in Py3.
- **`# encoding: utf-8` in Py3** — meaningless.
- **`if isinstance(x, basestring)`** — `basestring` doesn't exist in Py3. Use `str`.
- **Encoding everywhere "just in case"** — `.encode().decode()` chains hide real bugs and are slow.
- **Skipping production canary** — bytes/str bugs only manifest with real data.

## Verify it worked

- [ ] All tests pass on Python 3.12+.
- [ ] No `2to3`, `from __future__ import`, or Py2-specific guards remain.
- [ ] Bytes/str boundaries are explicit; internal code uses `str` only.
- [ ] All file I/O specifies mode (`"r"` text or `"rb"` bytes).
- [ ] Subprocess calls use `text=True` where output is treated as text.
- [ ] Divisions audited; `/` vs `//` matches intent.
- [ ] Dependencies bumped to Py3-compatible versions.
- [ ] No deprecation warnings on test run.
- [ ] CI runs against multiple Py3 minor versions (e.g. 3.11, 3.12).
- [ ] Production canary on Py3 for 1+ week with no error rate increase.
- [ ] All Py2 infra (shebang, base images, deploy scripts) updated.
