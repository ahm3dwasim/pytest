# OSS Report — pytest #14771

Issue: https://github.com/pytest-dev/pytest/issues/14771
`--pyargs` test discovery still fails in frozen application (e.g. PyInstaller).

## FLAG: SMALL FIX

The genuine bug fixable with a small, correct change is the **misleading error
message**, not making collection work from a frozen archive. pytest collection
is fundamentally file-system/path based; it has no collector that reads test
modules out of a PyInstaller/zip archive. That is an architectural "BIG WIN"
the maintainers already acknowledged does not exist (see #11511), and
emadleo13 confirmed on the issue that discovery already works once the test
files are bundled onto the file system (`datas=[("myapp","myapp")]`). So the
smallest correct, honest change is to replace the misleading
`missing __init__.py?` diagnostic with one that explains what actually
happened.

## Root cause (3 lines)

With `--pyargs`, `search_pypath()` calls `importlib.util.find_spec()` and, for
an importable package inside a frozen bundle, returns its `submodule_search_locations[0]`
(or `dirname(origin)`) — a path *inside the embedded archive* that does not
exist on the real file system. `resolve_collection_argument()` then fails the
`safe_exists(fspath)` check and raised the generic
`module or package not found: <arg> (missing __init__.py?)`, which is wrong and
confusing because the module *is* importable and *does* have an `__init__.py`.

## Diff summary

- `src/_pytest/main.py` — in `resolve_collection_argument()`, when `as_pypath`
  is set and `module_name is not None` (i.e. `search_pypath` found a spec and
  returned a path) but the path does not exist on disk, raise a dedicated
  `UsageError` explaining the module was found but its file-system path does
  not exist, and that this typically means a frozen/zipped application and
  pytest can only collect from files on the file system. The original generic
  messages are unchanged for the truly-not-found case.
- `testing/test_main.py` — added
  `TestResolveCollectionArgument::test_pypath_importable_but_not_on_filesystem`,
  which monkeypatches `importlib.util.find_spec` to simulate a PyInstaller-like
  frozen importer (spec resolving to a non-existent bundle dir) and asserts the
  new message.
- `changelog/14771.improvement.rst` — user-facing changelog entry.

The discriminator `module_name is not None` is reliable: `module_name` is set
only when `search_pypath` returned a concrete path (a spec was found). A
truly-missing module (`find_spec` → None) keeps the original
`missing __init__.py?` message, as verified by the existing `test_does_not_exist`.

## Test commands & output

Focused tests (editable install; `-o minversion=0` only because the dev
checkout's setuptools-scm version string trips pyproject's `minversion`):

```
$ python -m pytest testing/test_main.py -q -o minversion=0
25 passed in 0.32s

$ python -m pytest testing/test_collection.py -q -o minversion=0 -k "pyargs or pypath or namespace"
5 passed, 113 deselected in 0.89s
```

Regression test proven to catch the bug (fix temporarily reverted):

```
$ git stash push -- src/_pytest/main.py
$ python -m pytest testing/test_main.py::TestResolveCollectionArgument::test_pypath_importable_but_not_on_filesystem -q -o minversion=0
E   AssertionError: Regex pattern did not match.
E     Expected regex: 'module frozenpkg was found, but its file-system path does not exist'
E     Actual message: 'module or package not found: frozenpkg (missing __init__.py?)'
1 failed
$ git stash pop   # fix restored -> test passes
```

Lint/format:

```
$ ruff check src/_pytest/main.py testing/test_main.py
All checks passed!
$ ruff format --check src/_pytest/main.py testing/test_main.py
2 files already formatted
```

## Risks

- Low. Behavior change is limited to the error *text* on an already-failing
  path; no successful collection path is altered. The not-found branch and its
  existing test are untouched.
- The new branch triggers only when a spec was found but its path is missing on
  disk — frozen/zipped apps, or (rarely) a file deleted between `find_spec` and
  the existence check; the message is still accurate in that edge case.
- Does not make pytest collect tests from inside a frozen archive — that is out
  of scope and would be a large architectural change (a new archive-aware
  collector). Users should continue to bundle test files as data files.

## Draft PR description

> **Improve `--pyargs` error message for modules bundled in frozen/zipped apps (#14771)**
>
> When `--pyargs` is given a module that is importable but whose source files
> are not on the file system — typically inside a frozen/zipped application such
> as a PyInstaller bundle — pytest reported the misleading
> `module or package not found: <arg> (missing __init__.py?)`, even though the
> module is importable and has an `__init__.py`.
>
> `search_pypath()` resolves such a module to a path inside the embedded
> archive, which does not exist on disk, so `resolve_collection_argument()`
> failed the existence check with the generic "not found" message. This PR
> detects that case (a spec was found but its path does not exist) and raises a
> clearer `UsageError` explaining that the module was found but its file-system
> path does not exist, and that pytest can only collect tests from files
> available on the file system.
>
> This does not add collection from frozen archives (pytest collection is
> path-based); it makes the limitation discoverable instead of misdiagnosed as a
> missing `__init__.py`. Collecting from a frozen app still requires the test
> files to be bundled onto the file system.
>
> Added a focused regression test and a changelog entry.
>
> ---
> **AI assistance disclosure (per pytest's AI policy):** This change was
> prepared with AI assistance (Claude Code). Every commit ends with the
> `Co-authored-by: Claude <noreply@anthropic.com>` trailer. The repository
> owner has reviewed the change before submission.

## Estimated cost

Approximately **$2.0–2.5** of model usage for this session (reproduction,
code reading, fix, tests, lint, and report). Well under the $5 limit.
