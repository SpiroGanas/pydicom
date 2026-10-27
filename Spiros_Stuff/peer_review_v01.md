# pydicom — Peer Review v0.1

| | |
|---|---|
| **Reviewed commit** | `b218118` on `claude/epic-turing-86mhqm` (identical source to upstream `pydicom/pydicom@6f92f30`, "Drop references to Python 2 or 3 (#2375)") |
| **Package version** | `3.1.0.dev0` |
| **Review date** | 2026-10-27 |
| **Environment used** | Linux, Python 3.13.17, numpy 2.5.3, Pillow 12.3.0, pytest 9.1.1, ruff 0.15.12 / 0.16.9, mypy 1.20.2 / 2.3.1. The optional codec plugins (`pylibjpeg`, `pyjpegls`, `python-gdcm`) were **not** installed. |
| **Audience** | Spiro — first open-source contribution, aiming for senior-engineer-level contributions to this project |
| **Companion document** | `Spiros_Stuff/pydicom_contribution_report.md` (the earlier "where could I contribute" overview). This report goes a level deeper: concrete, reproduced defects and improvements, sorted by how hard they are to land. |

---

## 0. Executive summary

**State of the project.** pydicom is a mature, well-engineered library. The core reader/writer, the data model and especially the new `pydicom.pixels` package are cleanly designed, extensively documented in NumPy style, fully type-annotated (`mypy` is clean on the pinned version), formatted with `ruff format`, and covered by a large test suite (3,857 tests pass locally; upstream Codecov reports ~98.4 % line coverage with all plugins installed; 94.8 % was measured here without the codec plugins). Releases use PyPI trusted publishing, every GitHub Action is SHA-pinned, and dependabot/pre-commit.ci keep dependencies moving.

**Where it is weaker.** Almost everything a *new contributor touches first* has drifted: the `Makefile` has no working target, `CONTRIBUTING.md` still says `black` and contains a `pip` command that errors, 19 of the 21 doctest-able documentation files fail, ~100 documented outputs still show the pre-3.0 tag format, the `asv` benchmark suite no longer imports, and two "time bombs" are armed in the tooling (the ruff config inherits ruff's *default* rule set, which grew from 59 to 394 rules in ruff 0.16; mypy 2.x reports 13 new errors). In the code itself the defects cluster at the edges: duck-typing without a type check (`len(value)`, `hasattr(val, "append")`), copy-paste slips in lookup tables and messages, `assert` used for validation, error paths that tests never exercise, and a download/retry stack with no unit tests at all.

**Headline numbers from this review.**

| Metric | Value |
|---|---|
| Numbered findings in this report | 137 (several are batches of related one-line fixes), after removing items already tracked upstream |
| … of which reproducible code defects or inconsistencies | ≈ 70 |
| … documentation / docstring / markup defects | ≈ 45 |
| … test-suite and tooling improvements | ≈ 22 |
| Findings dropped because an upstream issue/PR already covers them | 6 full duplicates + 1 partial (listed in §1.3) |
| Local test run | 3,857 passed, 538 skipped (missing optional codecs), 2 failed (network-only tests) |
| Local line coverage | 94.8 % (13,425 / 14,168 statements) without codec plugins; upstream Codecov ≈ 98.4 % |
| Public functions with no direct test | ~25 (listed in §4.3) |

**If you only do five things, do these (each is a complete, mergeable PR):**

1. **`apply_color_lut(palette="FALL")` applies the WINTER palette and vice versa** — two swapped UIDs in `src/pydicom/pixels/processing.py:117-118`, plus a parametrised test over all eight palette names. *(P1, trivial, real user-visible bug)*
2. **`as_pixel_options()` tests `0x7FE00001` twice** instead of `0x7FE00001` and `0x7FE00002` (`src/pydicom/pixels/utils.py:241`) — one character, plus a test with only one of the two Extended Offset Table elements present. *(P3, trivial)*
3. **The test suite cannot run under `pytest-xdist`** because three `parametrize` decorators iterate over Python `set`s, so test ids differ between workers — wrap them in `sorted()`. *(T12, trivial; makes every contributor's test loop 3-4× faster)*
4. **The developer-setup docs are wrong in four places** (`black`, `pip install -e --group docs`, no mention of `pydicom-data`/`--group dev`, broken `Makefile`) — one docs PR that fixes the path every newcomer walks. *(D2–D4, T3, trivial/easy)*
5. **`PersonName.__eq__` and `sr.Code.__eq__` are not consistent with `__hash__`** — equal objects hash differently, so set/dict lookups silently fail. Two small, textbook fixes with obvious tests. *(C9 and M2, easy)*

---

## 1. Method, scope and what was excluded

### 1.1 What was done

1. **Static review of every module** in `src/pydicom/` (grouped as core data model; file I/O & encoding; pixel data; CLI/util/SR/data; docs & tooling), reading the code against its docstrings and the DICOM standard. **Every code finding below was reproduced by running a snippet against this checkout**; the exact command or snippet is given so you can repeat it. Findings that could not be reproduced were dropped.
2. **Tooling baseline** with the *pinned* versions from `.pre-commit-config.yaml` (`ruff 0.15.12`, `mypy 1.20.2`, `typos`) and with the current releases (`ruff 0.16.9`, `mypy 2.3.1`), to separate "broken today" from "breaks on the next autoupdate".
3. **Full test run with coverage** (`pytest tests --cov=pydicom`), serially and under `pytest-xdist`, plus the documentation doctests (`pytest doc --doctest-glob='*.rst'`).
4. **Upstream cross-check.** Every finding was searched against the issue and PR tracker of `pydicom/pydicom` (29 open issues, 40 open PRs, and closed items via keyword search). Items that an open or closed issue/PR already describes were **removed** (§1.3). Items that are *adjacent* to an upstream discussion are kept and carry a "related upstream" note so you can cite it in your PR.

### 1.2 Limitations you should know about

- The codec plugins (`pylibjpeg*`, `pyjpegls`, `python-gdcm`) were not installed, so 538 tests were skipped and the GDCM/pylibjpeg/pyjpegls plugin modules were only reviewed by reading. Coverage numbers for those modules are therefore lower here than on Codecov.
- GitHub's API was not reachable from the review environment; the upstream check used GitHub's HTML search. A few very old or unusually worded issues could have been missed — **always search the tracker again before opening an issue or PR**, and link what you find.
- This review was produced with AI assistance and then verified by running the reproducers. pydicom's PR template explicitly **rejects AI-only PRs**: for every item you take on, reproduce it yourself, understand the fix line by line, and write the PR summary in your own words.
- Line numbers refer to commit `b218118` and will drift as `main` moves.

### 1.3 Findings excluded because upstream already tracks them

| Finding | Upstream |
|---|---|
| `fileutil.reset_buffer_position` does not restore the position when the body raises | PR [#2410](https://github.com/pydicom/pydicom/pull/2410) (open) — same fix |
| `read_sequence_item` catches `BaseException` and re-raises everything as `OSError` | Issue [#2332](https://github.com/pydicom/pydicom/issues/2332) + PR [#2333](https://github.com/pydicom/pydicom/pull/2333) (open) |
| `apply_color_lut` wraps indices past 255 for an 8-bit-entry LUT with > 256 entries | Issue [#2402](https://github.com/pydicom/pydicom/issues/2402) + PR [#2403](https://github.com/pydicom/pydicom/pull/2403) (open) |
| `apply_modality_lut` fails with `uint12` for a 12-bit *LUT Descriptor* | Issue [#2317](https://github.com/pydicom/pydicom/issues/2317) + PR [#2326](https://github.com/pydicom/pydicom/pull/2326) (open) |
| HTTP error bodies (404) are written to the data cache as if they were the file | PR [#2390](https://github.com/pydicom/pydicom/pull/2390) (open) covers the status check; the two leftovers (custom `url` dropped on re-download, junk file left on disk) are kept as M7 |
| mypy 2.x reports 13 `bytes`/`bytearray` errors | PR [#2376](https://github.com/pydicom/pydicom/pull/2376) "Mypy 2.3" (open, draft; its author reports two `decoders/base.py` errors still unresolved — a place to help) |
| `pydicom codify` crashes with `TypeError: object of type 'NoneType' has no len()` on empty binary elements | Issue [#2391](https://github.com/pydicom/pydicom/issues/2391) + PR [#2392](https://github.com/pydicom/pydicom/pull/2392) (open) |

Also deliberately out of scope: the `# TODO: Remove in v4.0` deprecation removals (planned breaking changes), thread-safety of `config` (issue #2220 / PR #2300), new codecs (JPEG XL #2214, HTJ2K encoder #2156) and the other `codify` output problems already being fixed upstream (multi-valued AT/PN values, elements without keywords: issue #2391 / PRs #2392, #2399).

---

## 2. Repository snapshot

| Facet | Observation |
|---|---|
| Size | ~14,200 executable statements in `src/pydicom` (79 modules; 2,900 of them are generated dictionaries); 4,397 collected tests in 51 test modules (51,000 lines of tests) |
| Layout | `src/` layout, `flit_core` build backend, PEP 621 metadata, PEP 735 dependency groups (`dev`, `docs`), optional extras `basic`, `pixeldata`, `gpl-license`; zero mandatory runtime dependencies |
| Python support | 3.10–3.14 declared and tested (plus PyPy 3.10); upstream PR #2398 adds 3.15 |
| Activity (last 12 months, from the shallow clone) | 51 commits, 13 human authors; two maintainers account for ~40 % of commits; the rest is dependabot/pre-commit.ci and one-off contributors |
| Releases | 3.0.0 (Sep 2024) → 3.0.1 → 3.0.2 (Mar 2026, security fix for CVE-2026-32711, a DICOMDIR path-traversal) and a 2.4.5 backport; 3.1.0 has been "in progress" for over a year (`doc/faq.rst` still says "~February 2026") |
| Issue tracker | 29 open issues, 40 open PRs. Labels exist for `Difficulty-Easy/Medium/Hard` and `needs contributor`, but **no open issue carries them** and the `Difficulty-Easy` label was last used on issues closed years ago; there is no `good first issue` label at all. Expect to find your own work (which is what this report is for) rather than pick from a curated list |
| Contribution rules (from `CONTRIBUTING.md` / PR template) | Fork → feature branch → **draft** PR → "ready for review"; squash-merged; usually two approvals; bug fixes need a regression test that fails before the fix; new lines must be covered; type hints mandatory; NumPy-style docstrings; a release-note entry in `doc/release_notes/v3.1.0.rst`; **human-written PR summary, AI-only PRs are closed** |
| Standards | DICOM dictionary at 2024d (`src/pydicom/_version.py`); the standard is at <https://dicom.nema.org/medical/dicom/current/output/html/> — PS3.5 (encoding) and PS3.3 (IODs/modules) are the parts you will cite most |

---

## 3. How well does the repo use modern Python tooling?

**Overall grade: adequate — modern core, dated contributor surface.** The packaging, typing, formatting and release path are current; the things a contributor runs locally are stale or broken, and two configuration choices will fail the next automated tool bump.

### 3.1 Scorecard

| Area | Status | Evidence | Recommended change (difficulty) |
|---|---|---|---|
| Build backend & metadata | **Modern** | `flit_core>=3.12`, `src/` layout, PEP 621/639/735, single-sourced version (`_version.py` reads `importlib.metadata`), `py.typed` shipped | Remove the leftover `# dynamic = ["version"]` comment; move `types-pydicom` from the runtime `basic` extra to the `dev` group (trivial) |
| **uv** | **Not used** | No `uv.lock`, no `[tool.uv]`, no `uv` in CI, docs or scripts | Adopt: commit `uv.lock`; document `uv sync --group dev --extra basic && uv run pytest`; use `astral-sh/setup-uv` in `tests_wf.yml` and `build_doc.sh`. Benefits: reproducible docs builds (today Sphinx/theme resolve unpinned on every CircleCI run), one install recipe instead of three divergent ones, faster cached CI, and dependabot can bump the lockfile (medium — needs maintainer agreement) |
| **ruff (lint)** | **Adequate, with a time bomb** | `[tool.ruff.lint] extend-select = ["C90","PERF","E501","W","PL","UP"]`. `extend-select` *adds to ruff's default rule set*, which is version-dependent: 59 rules in the pinned 0.15.12, **394** in 0.16.9 (measured with `ruff check --isolated --show-settings`). Result: `ruff check src` is clean on 0.15.12 and reports 227 errors on 0.16.9 (678 across `src tests doc`). Upstream issue #2365 / PR #2368 are wrestling with exactly this bump — but #2368 keeps `extend-select` and only grows the `ignore` list, so the root cause stays | Replace `extend-select` with an explicit `select = ["E4","E7","E9","F","C90","PERF","E501","W","PL","UP"]` so the rule set no longer depends on the ruff version, then opt into new rules deliberately (easy; coordinate with #2368) |
| ruff scope | **Gap** | `include = ["src/*.py","tests/*.py","doc/*.py"]` excludes `examples/`, `util/`, `benchmarks/`, `build_tools/` (29 files never linted); the pre-commit `ruff-check` hook is further restricted to `files: src`, so `tests/` and `doc/` are linted by nothing. With the pinned ruff: `doc/conf.py:20` F401, `doc/conf.py:48` E402, `tests/pixels/test_encoder_pydicom.py:142` F841, `tests/test_fileset.py:834` F841, `tests/test_valuerep.py:691` PLW0177 — so the `ruff check .` that `CONTRIBUTING.md:174` tells you to run fails on a fresh clone | Fix the five violations, change the hook to `files: ^(src|tests|doc)/`, widen `include` (easy). Nobody upstream proposes lifting `files: src`: PR #2368 (ruff 0.16) keeps it and only adds `tests/*.py` per-file ignores, and PR #2310 merely drops the `check-symlinks` hook |
| Complexity gates | **Effectively disabled** | `max-complexity = 36`, `max-branches = 43`, `max-statements = 108`, `max-args = 17`; nothing trips them. With ruff's defaults (10/12/50) 49 functions exceed complexity, e.g. `dcmwrite` (`filewriter.py:1134`, complexity 30 / 33 branches / 94 statements), `_validate_options` (`pixels/common.py:724`, 23/22), `generate_fragmented_frames` (`encaps.py:199`, 20/23/63), `get_frame` (`encaps.py:448`, 20/22/68), `set_pixel_data` (`pixels/utils.py:1901`, 20/20/74) | Ratchet to 20/25/80 (flags only `dcmwrite` and `_validate_options`; add targeted `# noqa` with a refactor issue), then tighten as the top functions are split (medium) |
| ruff format | **Modern** | `ruff format --check` clean on all 135 files; `black` removed in #2299 | Delete the `black` instructions from `CONTRIBUTING.md:177-184` (trivial) |
| mypy | **Modern, with a time bomb** | `disallow_untyped_defs/calls`, `disallow_incomplete_defs`, `warn_return_any`; **clean on mypy 1.20.2**. mypy 2.3.1: 13 errors in 6 files (all `bytearray` no longer promoting to `bytes`) — tracked by PR #2376. Dead config: `exclude = ["src/pydicom/benchmarks/"]` (no such directory) and a `[[tool.mypy.overrides]]` for `pydicom.encoders.base`, a module that does not exist (the real one is `pydicom.pixels.encoders.base`). `warn_unused_ignores = false` lets stale ignores accumulate: 49 `# type: ignore` in `src/`, at least 8 of them unused (§5.2) | After #2376 lands: delete the dead `exclude`/override, enable `warn_unused_ignores = true` and `enable_error_code = ["ignore-without-code"]` and clean up (easy) |
| typos | **Good** | pre-commit hook, `_typos.toml` allow-list for DICOM tokens, generated dicts excluded; clean on `doc/`, `README`, `CONTRIBUTING`, `examples/` | Pin the hook to a concrete tag instead of floating `rev: v1` (trivial). Note `typos` does not catch its/it's — see D-group findings |
| pre-commit / pre-commit.ci | **Modern** | Standard hooks + typos + ruff + mypy; monthly autoupdate | The ruff and mypy bumps will fail until the two time bombs are defused |
| GitHub Actions CI | **Modern** | Reusable `tests_wf.yml`; matrix Windows 3.12/3.13/3.14, Ubuntu 3.10/3.11/3.14, macOS 3.10/3.11/3.12, PyPy 3.10; all actions SHA-pinned with version comments; `permissions: contents: read` everywhere; OIDC trusted publishing to PyPI | Add `concurrency` (cancel superseded PR runs) and `timeout-minutes`; add `reopened`/`ready_for_review` to the PR trigger; `apt-get update` before `apt install`; fix the "as of 2003-07" comment in `PR.yml:15` (easy). No workflow runs ruff/mypy/typos or builds the docs — only pre-commit.ci and CircleCI do |
| Coverage | **Good** | Codecov upload from one job per workflow; `.github/codecov.yml` sets a 0.2 % project threshold; `.coveragerc` omits `tests/*`, `benchmarks/*` | Move `.coveragerc` into `[tool.coverage.run]` and add `[tool.pytest.ini_options]` (`testpaths = ["tests"]`, `filterwarnings = ["error::DeprecationWarning:pydicom"]`) to `pyproject.toml` (easy) |
| Docs build | **Dated** | CircleCI config v2, TeX Live installed via `apt` on every run, `--force-yes`, deploys to `gh-pages` with a maintainer's personal deploy key (set up in 2017, issue #394) and hard-coded committer identity; no docs build on PRs in GitHub Actions; doctests fail (§3.2) | Move the build to GitHub Actions (artifact on PRs, `actions/deploy-pages` with OIDC on `main`), run `sphinx-build -W --keep-going` and a scheduled `linkcheck` (medium–hard; maintainer buy-in) |
| Benchmarks (`asv`) | **Abandoned** | `asv.conf.json` benchmarks branch `nested-seq-speed` (does not exist); not run anywhere; `bench_handler_numpy.py` and `bench_rle_encode.py` fail at import (`pack_bits` moved, `rle_encode_frame` removed); the rest import deprecated `pixel_data_handlers` | Either delete, or port to `pydicom.pixels` and add a manual `workflow_dispatch` job plus a CI step that at least imports every benchmark (easy–medium). Upstream PR #2405 reports asv numbers, so the maintainers do still use it ad hoc |
| Contributor entry points | **Broken** | `Makefile`: `test-code: pytest pydicom` (no such directory), `code-analysis: ruff .` (invalid), `clean` calls a non-existent `tools/rm_pyx_c_file.sh`, `test-doc: pytest doc/*.rst` fails on `doc/index.rst:60`; no `tox.ini`/`noxfile.py`/`justfile`; `CONTRIBUTING.md`, the two contributing tutorials and CI each give a *different* install recipe | One documented recipe (`pip install -e . --group dev`, needs pip ≥ 25.1, or `uv sync`), and either fix or delete the Makefile (easy) |
| Security / supply chain | **Good, with gaps** | Trusted publishing, SHA-pinned actions, least-privilege permissions, dependabot; **no `SECURITY.md`** although a CVE was shipped in 3.0.2; no `CODE_OF_CONDUCT.md`; no `CITATION.cff` although the README advertises a Zenodo DOI; the publish job builds from the tag without running tests | Add the three files (trivial; the policy text needs maintainer sign-off) |
| `.gitignore` | Stale | `tests/test_pixel.py`, `docs/_build/*` (directory is `doc/`), `pydicom.egg-info`, `.tox/*`, `distribute*`, `py3source` | Prune (trivial) |

### 3.2 Documentation toolchain specifics

- `python -m pytest doc --doctest-glob='*.rst'` → **19 failed, 2 passed**. Nine failures are illustrative snippets that can never run (`dcmread("path/to/dataset")`, missing imports) and should be marked `# doctest: +SKIP`; ten are genuinely stale outputs (D-group below).
- 104 code outputs in `doc/guides`, `doc/tutorials`, `doc/reference` still use the pre-3.0 `(gggg, eeee)` tag format (166 use the current `(gggg,eeee)`), so copied output no longer matches.
- `doc/conf.py:383-389` `doctest_global_setup` does `os.chdir(pydicom.__path__[0] + '/../tests/test_files')`, a path that does not exist under the `src/` layout — `make doctest` cannot work. `exclude_trees` (`:168`) is a pre-Sphinx-1.0 option name.
- `doc/README.md` tells you to install `sphinx_rtd_theme`/`sphinx_issues`; the real theme is `pydata_sphinx_theme` and the dependency list is the `docs` group.

---

## 4. Test coverage: what the numbers say and whether to work there

### 4.1 Numbers

| | |
|---|---|
| Full local run (serial) | `3857 passed, 538 skipped, 2 failed` in 3 min 08 s. The two failures (`tests/test_data_manager.py::test_fetch_data_files`, `::TestExternalDataSource::test_get_testdata_files_local_external_and_cache`) need network access |
| Line coverage, local, without codec plugins | **94.8 %** (13,425 / 14,168) |
| Line coverage, upstream Codecov, all plugins | **≈ 98.4 %** (quoted in the Codecov comment on PR #2310) |
| Modules at 100 % | 38 of 79; 57 of 79 at ≥ 95 % |

Per package (local run; low numbers for plugin packages are an artefact of the missing codecs):

| Package | Coverage | Statements | Comment |
|---|---|---|---|
| `pixel_data_handlers` (deprecated) | 66.0 % | 588 | GDCM/pylibjpeg/JPEG-LS handlers skipped |
| `data` | 82.9 % | 275 | `download.py` 71.7 %, `retry.py` 61.5 % — **real gap**, these need network-free unit tests (M20) |
| `pixels` | 89.3 % | 2,980 | plugin modules skipped; the pure-Python parts are ≥ 92 % |
| `overlays` | 90.7 % | 54 | |
| `util` | 91.9 % | 357 | `fixer.py` 65.9 % — **real gap** (M20) |
| `cli` | 95.8 % | 212 | |
| `waveforms` | 96.9 % | 65 | |
| top-level modules | 98.0 % | 6,727 | `fileutil.py` 72.4 % is the outlier — four documented helpers have no callers and no tests (I18) |
| `sr`, `encoders`, `examples` | 100 % | | |

### 4.2 Quality of the suite (not just quantity)

Strengths: scenario-driven tests with real reference files (`tests/pixels/pixels_reference.py`), good use of `parametrize`, assertions on warning messages, separate "missing library" tests.

Weaknesses that are worth fixing (each is a finding below):

- **`pytest -n` is unusable** (T12): three `parametrize` decorators iterate over sets, so test ids depend on `PYTHONHASHSEED` and xdist aborts with "Different tests were collected between gw1 and gw0".
- **Not offline-safe** (M22): 17 of the 85 files in `src/pydicom/data/urls.json` are not in the `pydicom-data` 1.0.0 package; without network, `get_testdata_file()` returns `None` and ~25 tests **fail** (`AttributeError: 'NoneType' …`) instead of skipping. Worse, if `pydicom-data` is not installed at all, `tests/pixels/*` fail at *collection* with `TypeError: argument should be a str or an os.PathLike object … not 'NoneType'`, and the contributing docs never mention `pydicom-data`.
- **290 `ResourceWarning`s** from `tests/test_fileset.py` fixtures that `return TemporaryDirectory()` and never clean up (M23).
- **One test takes 22 % of the suite's wall time** (`tests/pixels/test_utils.py::TestDecompress::test_too_long_raises`, ~19 s streaming 4 GiB) because `decompress()` checks the size limit only after materialising frames (M24).
- Tautological assertions in `tests/test_data_manager.py::test_get_dataset` (M21); 175 `pytest.raises(X)` without `match=`; a dead `chdir` helper and stale "# Fails" comments in `tests/test_util.py`; collection-time I/O in `tests/test_cli.py:94`.

### 4.3 Public functions with no direct test (confirmed by grepping `tests/`)

`Dataset.copy`, `Dataset.__array__`, `Dataset.elements`, `dataelem.empty_value_for_VR`, `valuerep.validate_pn_component`, `datadict.mask_match`, `values.convert_OVvalue`, `values.convert_UR_string`, `values.convert_SQ`, `fileutil.absorb_delimiter_item`, `fileutil.find_bytes`, `fileutil.find_delimiter`, `fileutil.length_of_undefined_length`, `util.fixer.fix_mismatch`, `util.fixer.fix_mismatch_callback`, `data.retry.retry`, `util.codify.code_file`, `data.download.get_config_dir`, `get_url`, `get_url_map`, `download_with_progress`, `data_file_hash_check`, `NoHashFound`, `data.data_manager.get_external_sources`, `online_test_file_dummy_paths`, `cli.show.quiet_rtplan` (only via `main`), `pixels.common.RunnerBase.frame_length(index=...)`, `pixel_data_handlers.util.apply_rescale` shim. Several of the bugs in §5 live exactly in these functions (`Dataset.copy`, `fix_mismatch`, `retry`, `download_with_progress`) — which is the point.

### 4.4 Is test coverage a good area for a new contributor?

**Yes — but targeted, not "raise the percentage".** The project is already at ~98 % upstream and the maintainers require coverage not to drop, so a PR whose only purpose is "+0.3 % coverage" is low value and may be declined. What *is* valuable, in order:

1. **Regression tests that accompany a bug fix** (every finding in §5 names the test to add). This is the single most-accepted kind of PR in this project's history.
2. **Tests for the untested public functions in §4.3**, especially `data/download.py`, `data/retry.py` and `util/fixer.py`, where behaviour is currently unverified and bugs were found.
3. **Test-infrastructure fixes** (xdist determinism, offline skips, fixture cleanup, the 19-second test) — small, universally appreciated, and they teach you pytest well.
4. Replacing `pytest.raises(X)` with `pytest.raises(X, match=...)` where the message is stable — do this opportunistically in files you are already touching, not as a sweep.

---

## 5. Findings, sorted by how easy they are for a new contributor

Each item gives the location (`path:line` at commit `b218118`), what is wrong, how to see it, the fix, and the test to add. IDs: **D** documentation, **T** tooling/tests, **C** core data model, **I** file I/O, **P** pixel data, **M** CLI/util/SR/data. "Related upstream" means an issue/PR touches the same code; read and cite it, it is not a duplicate.

### 5.1 First-time contributor changes (trivial)

These need no design discussion, touch one or a few lines, and come with an obvious test or none at all. **Batch the documentation ones into two or three PRs** (one per area) so reviewers can skim them; keep code fixes one-per-PR.

#### 5.1.1 Grammar, typos and stale text in the documentation

- **D1 — `README.md:1-2`** — the `unit-tests` and `type-hints` badges point at workflow names that no longer exist (current workflows are `PR`, `merge`, `tests_wf`, `publish-pypi-deploy`; nothing runs mypy on Actions). At review time the `unit-tests` badge rendered **"failing"** — a stale status from a long-deleted workflow. Replace with `actions/workflows/merge.yml/badge.svg` and the pre-commit.ci badge.
- **D2 — `CONTRIBUTING.md:237`** — `pip install -e --group docs` errors (`-e` needs a path); should be `pip install -e . --group docs` (as `doc/tutorials/contributing_docs.rst:32` already says). Introduced by PR #2298 (PEP 735 groups), which rewrote `pip install -e .[docs]`.
- **D3 — `CONTRIBUTING.md:177-184`** — still instructs `pip install black` / `black src/pydicom/`; the project uses `ruff format` since PR #2299 (which touched 56 files but not `CONTRIBUTING.md`; #2369 later removed the black config) and has no `[tool.black]`.
- **D4 — `CONTRIBUTING.md:147-151`, `doc/tutorials/contributing_code.rst:59-63`** — "`pip install pytest pytest-cov`" / "`pip install pytest` then `cd pydicom/tests; pytest`" omit the `dev` dependency group (`pydicom-data`, `pyfakefs`, …) that CI installs with `pip install --group dev .` (`tests_wf.yml:58`). Without `pydicom-data`, `tests/pixels/*` fail at collection (§4.2). Document `pip install -e . --group dev` and that it needs pip ≥ 25.1.
- **D5 — `CONTRIBUTING.md:233,240`** — reST `::` literal markers inside a Markdown file render as a literal double colon.
- **D6 — `CONTRIBUTING.md:226-231`** — claims bare `make` builds the docs "without the example gallery"; `doc/Makefile`'s default target is `help`. Point to `cd doc && make html`.
- **D7 — `CONTRIBUTING.md:205`** — "use the `pydicom_env_info` module" → `pydicom.env_info` (the command two lines later is right); `:215` "trackeback"; `:107` "to avoid GitHub to close" → "to prevent GitHub from closing".
- **D8 — `doc/README.md:8-22`** — obsolete toolchain (`mkvirtualenv`, `sphinx_rtd_theme`, `sphinx_issues`, a Docker image); the theme is `pydata_sphinx_theme` and the dependencies are the `docs` group.
- **D9 — `doc/tutorials/contributing_code.rst:48`** — ``See the `installation page</guides/user/installation>` `` lacks the `:doc:` role and renders as literal text.
- **D10 — `doc/tutorials/contributing_code.rst:36-46`** — tells new contributors to study `pydicom.pixel_data_handlers.pillow_handler`, deprecated since 3.0; point to `pydicom.pixels.decoders.pillow` and the plugin guides.
- **D11 — `doc/tutorials/contributing_code.rst:102-108,151`** — example links pinned to a 2020 commit with pre-`src/` paths ("around line 236" of `uid.py`; `RLELossless` is now at `src/pydicom/uid.py:343`).
- **D12 — `doc/tutorials/contributing_code.rst:218`, `doc/tutorials/contributing_docs.rst:210`** — "Your recently push branches" → "pushed"; `contributing_docs.rst:29` "dependencies requires for" → "required".
- **D13 — `doc/tutorials/contributing_docs.rst:105-118`** — the sample toctree lists `installation` and `virtualenvs`, tutorials that do not exist; copy the real one from `doc/tutorials/index.rst:12-19`.
- **D14 — `doc/release_notes/v3.1.0.rst:52-53`** — "better suited then" → "than"; `:49` missing full stop; `:54` "Fixed fileset-uid-property-annotation with Python 3.14" reads like a branch name; the blank line at `:62` splits the Fixes list.
- **D15 — `doc/release_notes/v3.1.0.rst:68` vs `doc/faq.rst:56`** — release notes say "Python 3.12, 3.13 and 3.14 are now supported" but the FAQ lists 3.12 under 3.0. Should be "3.13 and 3.14".
- **D16 — `doc/faq.rst:58`** — "3.1 … ~February 2026" has passed; say "unreleased (dev)" until the release.
- **D17 — `doc/release_notes/v3.1.0.rst`** — commit `bb171d1` "Dunder comparison operators return bool (#2374)" changed public `__eq__`/`__ne__` return types in five modules but has no release-note entry.
- **D18 — `doc/index.rst:80`** — DICOMDIR example shows `'9890234'`; the real value is `'98890234'` (`doc/guides/user/quick_start.rst:288` has it right).
- **D19 — `doc/guides/user/private_data_elements.rst:59`** — `d[0x00091001]` uses an undefined name; everything else on the page calls it `ds`.
- **D20 — `doc/guides/user/base_element.rst:28-29`** — `print(path)` shown as `PosixPath('…')`; `print` of a `Path` prints the bare string, and the path predates the `src/` layout.
- **D21 — 104 lines across `doc/guides`, `doc/tutorials`, `doc/reference`** — `(6000, 3000)` style outputs should be `(6000,3000)`; e.g. `working_with_overlays.rst:22`, `tutorials/filesets.rst:79-85`, `guides/cli/cli_intro.rst:15-16`, `private_data_elements.rst:89-90,110-111`. Regex-replace inside code blocks only, check prose by hand. (easy rather than trivial because of the volume)
- **D22 — stale doctest outputs** — `doc/tutorials/waveforms.rst:66` and `doc/guides/user/working_with_waveforms.rst:30` (`SamplingFrequency` now reprs `'1000'`); `doc/tutorials/sr_basics.rst:37` and `doc/reference/sr.rst:34` (CID keyword lists changed: `'SpecificObservationSubjectClass'` missing, `EchoFindingObservationTypes` → `…Type`); `doc/guides/user/working_with_overlays.rst:22` (`examples.overlay` has 18150 elements, not 29282); `doc/guides/user/working_with_pixel_data.rst:64` (NumPy 2 prints `shape=(64, 64)`, as `doc/index.rst:103` already shows); `doc/guides/encoding/rle_lossless.rst:119` and `doc/tutorials/pixel_data/introduction.rst:276` are environment-dependent and need `+SKIP`/`+ELLIPSIS`.
- **D23 — `doc/guides/cli/cli_dev.rst:7,11,15-27,84`** — tells plugin authors to edit `setup.py` `entry_points`; pydicom has no `setup.py`, the sub-commands live in `pyproject.toml:66-68` under `[project.entry-points.pydicom_subcommands]`.
- **D24 — `.github/ISSUE_TEMPLATE/pixel_issue.md:11,34-41`** — references the deprecated `pydicom.pixel_data_handlers.convert_color_space()` and `from pydicom.util import debug_pixel_data`, a function that does not exist anywhere in `src/` (the snippet raises `ImportError`). `bug_report.md:22` still says "(available in pydicom 2 or later)".
- **D25 — `examples/README.txt`** — the gallery landing text is literally "Somewhere to start".
- **D26 — `doc/guides/user/installation.rst:42-45`** — offline-install example names `pydicom-3.0.1-py3-none-any.whl`, a release with a known CVE; use a version-agnostic placeholder.
- **D27 — `doc/guides/user/viewing_images.rst:11,14`** — links to the Python 3.4 tkinter docs and an `http://` dclunie page.
- **D28 — duplicated words** — `doc/tutorials/pixel_data/compressing.rst:35` "for the the", `doc/guides/user/quick_start.rst:160` "pixel data data".

#### 5.1.2 Docstring markup that silently does not render (Sphinx/numpydoc)

A single colon (`.. note:`) turns a directive into a **comment**; a missing space (`..versionadded::`) turns it into **plain text**. Grep for `^\s*\.\.(version|note|code)` to find them all. Confirmed locations:

- **D29** — `src/pydicom/dataset.py:1273` (`.. versionchanged: 3.0`), `:2455` and `:3024` (`..versionchanged:: 2.0`), `src/pydicom/uid.py:474` (`.. versionadded: 3.0`), `src/pydicom/filereader.py:1117` (`.. note:`), `src/pydicom/misc.py:57` (`..versionadded:: 3.1`), `src/pydicom/data/download.py:50,80,151,220,314` and `src/pydicom/data/data_manager.py:165` (`..versionchanged:: 3.1`), `data_manager.py:209` (`.. versionchanged: 2.1`), `data_manager.py:13` (`.. codeblock: python` — the whole "external data sources" example is invisible in the rendered docs), `src/pydicom/sr/coding.py:13` (`..versionadded: 1.4`), `examples/image_processing/reslice.py:9-22` (`.. usage:` and `.. note:`).
- **D30** — `src/pydicom/data/data_manager.py:345-346,418-419` — `Raises` underlined with `______`; numpydoc only recognises `------`, so the `Raises` entries are glued onto `Returns`. `src/pydicom/util/hexutil.py:18` — `::` immediately followed by the indented block produces "Definition list ends without a blank line".
- **D31** — broken Sphinx roles: `src/pydicom/dataelem.py:336,424` ``:class`data element`` (missing colon after `:class`); `src/pydicom/sr/codedict.py:154` unclosed double backticks around `"FontanelOfSkull"`; `src/pydicom/dataset.py:3710` and `src/pydicom/fileset.py:352-353` use the heading `Params` instead of `Parameters`; `src/pydicom/util/codify.py:506-507` `Parameters:` with a colon and an over-long underline.

A ~10-line check (`rg -n "^\s*\.\.(version|note|code)" src doc examples`) added to `pre-commit` would stop these recurring — a nice follow-up PR once the fixes are in.

#### 5.1.3 Docstrings that disagree with the code (one-line fixes)

- **C15** — `src/pydicom/dataset.py:559-570` `Dataset.data_element()` says it returns `None` when the element is absent; `Dataset().data_element("PatientName")` raises `KeyError` (only a non-keyword name returns `None`). Document the `KeyError` (changing behaviour would be an API change).
- **C16** — `dataset.py:949-961` `original_character_set` promises `'iso8859'` for new datasets; `repr(Dataset().original_character_set)` is `''` and `__getitem__` relies on it being falsy. Fix the docstring.
- **C17** — `dataset.py:3532-3536` `validate_file_meta` documents `ValueError`; it raises `AttributeError` (and the tests assert that). Fix the docstring.
- **C18** — `dataset.py:1659-1661` `Dataset.setdefault` names `reading_validation_mode`; the code at `:1675` uses `writing_validation_mode` (verified by flipping the two modes).
- **C21** — `dataelem.py:63-64` `empty_value_for_VR` refers to `config.use_none_as_empty_value`, which does not exist (`use_none_as_empty_text_VR_value` does); `datadict.py:448-449` `keyword_for_tag` says group-length elements "will always return 'GroupLength'" but `keyword_for_tag(0x00080000)` is `''`; `datadict.py:57-68` `add_dict_entry` omits the `keyword` parameter; `dataset.py:902` `Dataset.values` says "Returns dict_keys".
- **I19** — `src/pydicom/filebase.py:187-189` `read_exact` says `None` reads everything but `read_exact(None)` raises `TypeError`; `filebase.py:301` documents `buffer` but the parameter is `initial_bytes`; `filebase.py:115,142` error text `"…is_little_endian' has not been set"` has an unbalanced quote; `fileutil.py:140-141` `read_undefined_length_value` "Returns delimiter" (it returns the value); `filewriter.py:858` `write_sequence` documents `data_element` (parameter is `elem`); `filewriter.py:351` `write_numbers` "*from* the dicom file" (it writes to it); `charset.py:499-503` `encode_string` mentions `reading_validation_mode`, code uses `writing_validation_mode`; `filereader.py:742` `read_preamble` says position 133 (it is 132); `misc.py:72-73,101`; `jsonrep.py:119,318`.
- **P7** — `src/pydicom/pixels/utils.py:1959-1960` `set_pixel_data` promises arrays up to 4294967294 bytes; the check at `:2004-2010` is `arr.nbytes > 2**31 - 2`. `decompress()` (`:694`) and `encaps.py` use the real `2**32 - 1` bound. Decide which limit is intended (ask in an issue), then align docstring and code (and add a test using a `np.broadcast_to` view, which has a huge `nbytes` without allocating).
- **P18** — wrong tags/UIDs in messages: `pixels/encoders/base.py:564` error says `(0028,0006) Photometric Interpretation` (that tag is *Planar Configuration*; PI is `(0028,0004)`); `encoders/base.py:992` lists *Deflated Image Frame Compression* as `1.2.840.10008.1.8.1` (actual `1.2.840.10008.1.2.8.1`); `pixels/utils.py:1660,1673` call *Bits Allocated* `(0028,0101)` (that is *Bits Stored*); `utils.py:2129` `unpack_bits` says `(0028,0011) Bits Allocated` (that is *Columns*); `pixels/common.py:195` `remove_plugin` message has a stray trailing quote (tighten the `match=` in `tests/pixels/test_common.py:624` so it would have caught it).
- **P19** — "it's" for "its" in `pixels/utils.py:1143,1209,1478,1542`, `pixels/decoders/base.py:1276,1316,2067,2105`, `pixels/common.py:698`, `dataset.py:2272,2314`; "an native" `common.py:157`; `decoders/base.py:1821,1934` have an extra `"` after the opening `"""`; `utils.py:1081-1083` `get_nr_frames` mentions an `arr` parameter it does not have; `utils.py:254/350` `compress` types `transfer_syntax_uid` as `UID` but the signature says `str`.
- **M17/M18** — `src/pydicom/data/download.py:157` documents `path` (parameter is `config_dir`); `download.py:284-296` `get_cached_filehash` describes calculating a hash but only reads `hashes.json` and raises an undocumented `NoHashFound`; `util/fixer.py:65,68` (`list` vs `tuple[str, ...]`, `process_unknown_VR` vs `process_unknown_VRs`); `util/codify.py:88-103` `include_private` undocumented; `util/leanread.py:90-92` promises a 7-tuple, yields a 5-tuple; `src/pydicom/examples/__init__.py:41-49` — `get_path` is documented in `doc/reference/examples.rst:45` as a function but only exists via module `__getattr__` (`'get_path' in dir(pydicom.examples)` is `False`, so IDEs and type checkers cannot see it): define it as a real function.
- **C26 (batch)** — `dataelem.py:116` "While its possible" / `:124` "Its far more" / `:158` "it's value" / `:201` "an int or int" / `:151` `0xFFFFFFFFL` (Python-2 long suffix missed by #2375) / `:756` "unscriptable"; `uid.py:55-56` the two comments `# Explicit VR Little Endian` and `# Implicit…` are **swapped** (`1.2.840.10008.1.2` *is* Implicit); `uid.py:156` "encasulated"; `uid.py:42-43` example output; `valuerep.py:1738` "decoded by" → "encoded"; `:1938` `name_suffix` described as "prefix"; `:1442,1471` "encodings uses"; `:2034` link anchor should be `sect_6.2.1.2`; `datadict.py:162` "2 high bytes" (it is one byte); `dataset.py:9,14` "(dict subclass)"/"(list subclass)" are false; `:2507` double comma; `tag.py:113` double space in an error message; `values.py:507` missing quote; `:720` "Returns bytes or str" (always `str`); `multival.py:2`.
- **I20 (batch)** — `errors.py:8` "the the"; `dicomio.py:2` "Many point of entry" → "Main"; `fileutil.py:274,293` "or or"; `charset.py:369` "it's", `:686` "If a list if passed", `:746` "other the"; `config.py:351` "enably", `:383` "is is", `:384` "en empty", `:201` the `IS`-as-ndarray sentence contradicts its own default; `filereader.py:153,858,1015`; `encaps.py:81-82,153,290,337,361-362,562,751,835`; `fileset.py:83-84` (`8 - prefix` should be `8 - len(prefix)`), `:1355,1388`; `jsonrep.py:148,221` "Return a the".
- **M19** — `src/pydicom/util/hexutil.py:31-32` still explains Python 2 behaviour (missed by #2375); `src/pydicom/cli/main.py:28-29,92` justify a syntax restriction with `eval`, which the CLI no longer uses; `src/pydicom/cli/codify.py:27` "Codify existed before as a stand-alone before"; `src/pydicom/sr/codedict.py:127` "an sorted".

#### 5.1.4 One-line code fixes (each with a 3–10 line regression test)

- **P1 — swapped palette UIDs.** `src/pydicom/pixels/processing.py:117-118` maps `"FALL"` → `…1.5.8` and `"WINTER"` → `…1.5.7`; the standard (and `_uid_dict.py:74-75`, and the `datasets` dict four lines above) have Fall = `.5.7`, Winter = `.5.8`. `apply_color_lut(arr, palette="FALL")` therefore applies the Winter palette. Repro: compare `apply_color_lut(arr, palette="FALL")` with `apply_color_lut(arr, ds=dcmread(get_palette_files("fall.dcm")[0]))`. Fix: swap the two strings (or build the dict from `pydicom.uid`). Test: parametrise over all eight names in `tests/pixels/test_processing.py` (only `"PET"` is tested today, lines 567-815).
- **P3 — duplicated tag check.** `src/pydicom/pixels/utils.py:241` `if 0x7FE00001 in ds._dict and 0x7FE00001 in ds._dict:` — second operand should be `0x7FE00002`. A dataset with *Extended Offset Table* but no *Extended Offset Table Lengths* raises `AttributeError: 'Dataset' object has no attribute 'ExtendedOffsetTableLengths'` from every `pixel_array` call. Test: dataset with only one of the two elements → no `extended_offsets` option.
- **P4 — `intent=0` ignored.** `processing.py:1229` `intent = intent if intent else ImageCms.getDefaultIntent(profile)` treats the documented value `0` (perceptual) as "not given"; use `if intent is None`. Also `_CMS_INTENTS[0]` at `:35` is spelled `"procedural"` (should be `"perceptual"`). Test: patch `ImageCms.buildTransform` and assert the rendering intent passed for 0..3 (only `-1` is tested, `tests/pixels/test_processing.py:2161,2263`).
- **P8 — `pack_bits` mutates its input.** `pixels/utils.py:1380-1382` `arr += b"\x00" * …` extends a caller's `bytearray` in place on the no-numpy path. Convert to `bytes` before padding; assert the input length is unchanged in `test_pack_bytearray` (`tests/pixels/test_utils.py:1368`).
- **P14 — `assert` as control flow.** `pixels/utils.py:165-172` `_array_common()` relies on `assert len(data) == 8` inside `try/except` to detect a missing *Pixel Data* element; under `python -O` the user gets `struct.error: unpack requires a buffer of 8 bytes` instead of the friendly `AttributeError`. Replace with an explicit `if`.
- **P15 — broken deprecation shim.** `src/pydicom/pixel_data_handlers/util.py:79-88` `_DEPRECATED` lacks `apply_rescale`, so `from pydicom.pixel_data_handlers.util import apply_rescale` (valid in 2.x) raises `ImportError` instead of the intended `DeprecationWarning` (`pixel_data_handlers/__init__.py:24` has it). Add the entry and a test in `tests/test_handler_util.py`.
- **P17 — wrong plugin name in metadata.** `src/pydicom/pixels/encoders/pyjpegls.py:33` records `encoding_plugin="pylibjpeg"` (copy-paste). Change to `"pyjpegls"`; assert `runner.get_frame_option(0, "encoding_plugin")` in a test (stub `jpeg_ls.encode_pixel_data` if the plugin is not installed).
- **C4 — "Private Creator" for the wrong elements.** `src/pydicom/dataelem.py:785-786` tests `self.tag.element >> 8 == 0`, true for `0x0000`–`0x00FF`; private creators are `0x0010`–`0x00FF`, `(gggg,0000)` is a group length and `0x0001`–`0x000F` are reserved. `DataElement(0x00090000, "UL", 100).name` → `'Private Creator'`. Use `self.tag.is_private_creator` and return `"Group Length"` for element 0 (mirroring the public branch at `:794-795`). Related upstream: PR #1350 fixed a recursion for the same reserved elements but left the name.
- **C8 — unpadded year.** `src/pydicom/valuerep.py:691` `DA(datetime.date(999, 1, 1)).original_string` → `'9990101'` (7 chars, invalid); `DT` at `:801` already uses `{val.year:04}`. One-character fix plus a test.
- **C14 — `convert_tag` ignores `offset`.** `src/pydicom/values.py:96-99` checks `len(byte_string) < 4` but not `offset + 4`; `convert_tag(b"\x08\x00\x10\x00", True, offset=2)` raises `struct.error` instead of the documented `ValueError`. Related upstream: PR #1833 introduced the check.
- **C22 — wrong exception type.** `src/pydicom/dataset.py:3667-3677` `FileMetaDataset.__setitem__` reads `value.tag` before delegating, so `FileMetaDataset()[0x00020010] = "1.2.3"` raises `AttributeError` where `Dataset` raises the clear `TypeError("Dataset items must be 'DataElement' instances")`. Check `isinstance` first.
- **I6 — missing `Z`.** `src/pydicom/misc.py:116` the `letters` string ends at `…XY`, so `find_keyword_candidates` (used for "did you mean" suggestions in `Dataset.__setattr__`) can never suggest `ZoomFactor`, `ZoomCenter`, …: `find_keyword_candidates("oomFactor")` → `[]`. Use `string.ascii_letters`; add the case to `tests/test_misc.py`.
- **I13 — required `ds` kwarg.** `src/pydicom/hooks.py:283,346` `raw_element_value_fix_separator` / `raw_element_value_retry` declare `ds: "Dataset | None"` without a default, unlike the default callback (`:160`) and the `RawDataHook` protocol (`:26`); calling them the documented way raises `TypeError: missing 1 required keyword-only argument: 'ds'`. Add `= None`.
- **I12 — examples that raise.** `hooks.py:307-310,368-371` examples call `register_kwargs("raw_element", …)`; the only accepted name is `"raw_element_kwargs"` (`hooks.py:115-118`) → `ValueError: Unknown hook 'raw_element'`. Also `config.py:115-118` tells users to migrate to `Hooks.reset`, which does not exist. Fix the examples; either add a `Hooks.reset()` (easy, with a test) or fix the reference.
- **I14 — error names the wrong element.** `src/pydicom/jsonrep.py:285-286` uses `self.tag` (the parent SQ) instead of `key` (the nested element lacking `"vr"`), so the user is pointed at `00081140` when `00081155` is at fault. Extend `tests/test_json.py`.
- **I15 — inconsistent EOF behaviour.** `src/pydicom/filebase.py:230-236` `read_UL`/`read_US` use `read()` and raise `struct.error` on a short read, while `read_tag` (`:223-228`) uses `read_exact` and raises `EOFError`; `encaps._read_item` (`encaps.py:1703-1709`) only catches the latter. Use `read_exact`. Related upstream: PR #2394 fixes the same class of bug in `filereader.data_element_generator` but not here.
- **M8 — `--exclude-size 0`.** `src/pydicom/util/codify.py:131` `if exclude_size:` treats `0` as "no limit", so `pydicom codify -e 0 file.dcm` emits the full pixel data instead of excluding every non-empty binary value. Use `is not None`. Note: open PR #2392 edits the very next line (the `None` crash) but leaves this truthiness test — build on top of it or mention it in your PR.
- **M11 — `util/dump.py`.** `:120` the recursive `pretty_print(dataset, indent_level + 1)` drops `indent_chars`; `:78` `hexdump` compares the line *start* with `stop_address`, so the line starting at `stop_address` is still printed and `stop_address=0` means "no limit".
- **M3 — `Code == None` raises.** `src/pydicom/sr/coding.py:40` accesses `other.scheme_designator` without a type check: `Code(...) == None` → `AttributeError`, and `code in [None, "x"]` blows up. Return `NotImplemented` for non-`Code` operands. (`Code.__new__.__defaults__ = (None,)` at `:65` is also redundant with the `NamedTuple` annotation.) Related upstream: #1994/#1995 made `Code == Code` return `False` instead of raising for unmapped SRT codes; the non-`Code` operand case was not covered.
- **M19b — `env_info` omits `rle`.** `src/pydicom/env_info.py:23-32` reports `pylibjpeg`, `openjpeg`, `libjpeg` but not `rle` (pylibjpeg-rle), which the handlers import; bug reports therefore omit it. Add `"rle"` and update `tests/test_env_info.py::test_all_modules_reported`. (PR #1349 added the other three pylibjpeg packages; `rle` was never added.)
- **C28 — dead code.** `src/pydicom/dataset.py:2419-2426` excludes `"description"` in `formatted_lines`, an attribute `DataElement` no longer has; `valuerep.py:1109,1414` "Issue #937 (Python 3.8 compatibility)" comments justify `repr(self)[1:-1]`, replaceable by `float.__repr__(self)`/`int.__repr__(self)` on 3.10+. Related upstream: #937/#938 introduced it.
- **I17 — dead or redundant code.** `fileutil.py:522-524` `while chunk := buffer.read(n): if chunk:` (always true); `fileset.py:382` `if path is not None` right after `path = Path(...)`; `fileset.py:1287` a trailing `continue`; `fileset.py:1273-1280` and `:2150-2157` identical size-guard blocks (hoist into a helper); `filereader.py:1157-1158` `is_filename = isinstance(...)` immediately followed by the same `isinstance`; `size_in_bytes(defer_size)` applied twice (`filereader.py:1086` and `:131`).
- **P23 — accidental writes to the dataset.** `src/pydicom/pixels/processing.py:568,765,786-787` do `ds.VOILUTSequence = cast(...)`, `ds.BitsStored = cast(...)`, `ds.RescaleSlope = …` purely to satisfy mypy, but these go through `Dataset.__setattr__` and replace the element value objects of a dataset the function is documented to only read. Assign to locals instead.
- **P24 — duplicated/missing logic.** `src/pydicom/waveforms/numpy_handler.py:100-141` `multiplex_array()` is a verbatim copy of the loop body of `generate_multiplex()` (`:175-216`); `pixels/utils.py:1433` passes a meaningless second argument to `importlib.import_module`; `pixels/utils.py:1846-1860` `reshape_pixel_array` hard-codes the JPEG-family UIDs that force *Planar Configuration* 0 but omits the three HTJ2K UIDs added in 3.0.
- **T13 — collection-time `TypeError` without `pydicom-data`.** `tests/pixels/pixels_reference.py` builds `Path(get_testdata_file(...))` at import; when the file is unavailable `get_testdata_file` returns `None` and `tests/pixels/test_decoder_*.py`, `test_utils.py` **fail at collection** with `TypeError: argument should be a str or an os.PathLike object … not 'NoneType'`. Make `PixelReference.path` tolerate `None` (and skip, see M22) and document `pydicom-data` in the contributing docs (D4). Related upstream: #1800 (closed as stale in 2025) reported the older runtime form of this (`dcmread: Expected a file ... but got NoneType`); its thread proposed skipping without `pydicom-data` and adding it to the dev extras — the latter happened (`dev` group), the docs never followed.

#### 5.1.5 Tests to add (no production change)

- **C27** — `Dataset.copy()` isolation (see C11 for why it matters), `Dataset.elements()` yields raw elements, `numpy.asarray(ds)`, `validate_pn_component` length limit, `mask_match` for a `60xx`/`50xx` tag and a private tag, direct `convert_UR_string`/`convert_OVvalue` padding behaviour.
- **I18** — `fileutil.absorb_delimiter_item`, `find_bytes`, `find_delimiter`, `length_of_undefined_length`: public, documented in `doc/reference/fileio.util.rst`, **zero callers and zero tests**. Either test them (little/big endian, delimiter spanning a `read_size` boundary, `rewind=False`, not-found returns `None` — which the docstring, claiming `int`, does not mention) or open an issue proposing deprecation.
- **P25** — `apply_color_lut` with each palette *name*; `create_icc_transform` intents 0..3; `as_pixel_options` with only one Extended Offset Table element; `RunnerBase.frame_length(index=...)` (new in 3.1, only tested indirectly); `get_j2k_parameters` with `LBox == 0/1` and truncated boxes; `EncodeRunner._validate_buffer` with multi-frame 1-bit input.
- **M20** — `util.fixer.fix_mismatch` (mirror `TestDataElementCallbackTests`), `data.retry.retry`, `util.codify.code_file` (tests call `main` instead), and the `data/download.py` helpers with `requests.get`/`urlretrieve` monkeypatched.
- **M21** — `tests/test_data_manager.py:63-64` computes `bases` under "Test that top level file is included" and asserts nothing; `:72-86` `continue`s on `not (A or B or C)` and then asserts `A or B or C` (always true). Decide what the test should check and assert it.
- **M23** — `tests/test_fileset.py:83-95,104-107,394-403` fixtures `return TemporaryDirectory()`; 290 `ResourceWarning: Implicitly cleaning up …` per run. `yield` and `cleanup()`, or use `tmp_path`.
- **T12 — make `pytest -n` work.** `tests/test_dataelem.py:674` (`STR_VR - ALLOW_BACKSLASH - {VR.UR}`), `:1485` and `tests/test_filewriter.py:3331` (`BUFFERABLE_VRS`) parametrise over sets; wrap in `sorted()`. Verify with `PYTHONHASHSEED=1 pytest --collect-only -q` vs `PYTHONHASHSEED=2` (28 differing lines today), then `pytest -n 4`. Related upstream: PR #2237 (merged 2025) set out to "add support for pytest-xdist", hit a *different* blocker (workers overwriting each other's downloads, solved with pyfakefs) and never saw this one.
- **Smaller test smells** — `tests/test_util.py:52-61` unused `chdir` helper; `:123-131` a "# Fails" comment on a case that now passes; `:161,363` docstrings name the wrong function; `tests/test_cli.py:94` runs `filespec_parser` at collection time (a parse error breaks the whole module — make it a fixture); `tests/test_examples.py:42` `assert ct is ct`.

#### 5.1.6 Trivial tooling fixes

- **T5** = D1 (badges). **T3** — fix or delete the `Makefile` (`test-code: pytest tests`, `code-analysis: ruff check . && ruff format --check .`, drop the `.pyx` line and the `tools/` reference). **T11** — delete the dead mypy `exclude` entry and the `pydicom.encoders.base` override. **T6** — add `SECURITY.md` (ask the maintainers how they want reports routed; GitHub private advisories were used for CVE-2026-32711), `CODE_OF_CONDUCT.md`, `CITATION.cff`. `.gitignore` pruning; pin the `typos` hook; `# dynamic = ["version"]` comment; `PR.yml:15` "2003-07"; `tests_wf.yml:42` `apt-get update`; `.circleci/config.yml` → version 2.1 and drop `--force-yes`.
- **M15** — two `benchmarks/` modules fail to import (`bench_handler_numpy.py:15-19` `pack_bits` moved to `pydicom.pixels.utils`; `bench_rle_encode.py:6` `rle_encode_frame` no longer exists); the others use deprecated `pixel_data_handlers`, `decode_data_sequence`, `itemise_frame`, `is_little_endian`, `write_like_original`. Port to `pydicom.pixels` or propose deleting (see T7).
- **M16** — gallery examples: `examples/metadata_processing/plot_add_dict_entries.py:28-29` and `examples/memory_dataset.py:40` set `Dataset.is_implicit_VR`/`is_little_endian` (fail under `-W error::DeprecationWarning`); `examples/image_processing/plot_downsize_image.py:6-8,23-24` says "512x512 to 64x64" and "is store[d] as a dicom image" but uses the 64×64 `examples.mr` and saves nothing; `reslice.py:9-22` single-colon directives (D29); grammar in `plot_dicom_difference.py:6`, `plot_add_dict_entries.py:6`, `plot_sequences.py:6,13`, `plot_anonymize.py:55,70`.

### 5.2 Early-contributor changes (easy)

Small code changes (5–40 lines) where the right behaviour is clear but you must write a proper regression test and a release-note line, and sometimes touch two places consistently.

- **C9 — `PersonName` hash/eq contract.** `src/pydicom/valuerep.py:1672-1674` `__eq__` returns `str(self) == other` (so `PersonName("X") == "X"`), but `__hash__` (`:1700-1702`) returns `hash(self.components)`: `pn in {"CITIZEN^Joan"}` → `False`, `{"CITIZEN^Joan": 1}.get(pn)` → `None`. Fix: `return hash(str(self))`. Release note needed (hash values change). Related upstream: #784/#785 made the hash value-based in 2019 but did not consider `str` equality.
- **M2 — `sr.Code` hash/eq contract.** `src/pydicom/sr/coding.py:21-22` hashes `scheme_designator + value` while `__eq__` (`:24-59`) maps legacy `SRT` codes to `SCT` first (and `tests/test_codes.py::test_equal_equivalent_coding` relies on that): `Code("T-B7000","SRT","m") == Code("111002","SCT","m")` is `True`, their hashes differ, `srt in {sct}` is `False`. Factor the SRT→SCT normalisation into a helper used by both methods. Related upstream: #1434 added `__hash__`; #1994/#1995 fixed a `KeyError` in `__eq__` for unmapped SRT codes — neither considered the hash contract.
- **M4 — `sr.codedict` objects cannot be copied or pickled.** `src/pydicom/sr/codedict.py:147-161,316-332` `__getattr__` dereferences instance attributes (`self.name`, `self._collections`); `copy.copy(codes.SCT)`, `copy.deepcopy(codes.CID2)`, `pickle.dumps(codes)` all hit `RecursionError` because `copy`/`pickle` create the instance via `__new__` and the first `__setstate__`/`_name` lookup recurses. Guard with `if name.startswith("_"): raise AttributeError(name)`; add copy/pickle tests.
- **C1 — `len()` on a non-sized `UN` value.** `src/pydicom/dataelem.py:227-236` calls `len(value) < 0xFFFF` before the setter runs; `DataElement(0x00280010, "UN", io.BytesIO(b"\x00\x01"))` and `DataElement(tag, "UN", 5)` die with a bare `TypeError` instead of the setter's descriptive `ValueError`. Guard with `isinstance(value, bytes | bytearray)`.
- **C2 — `DataElement.__eq__` can raise, return a non-bool, and is asymmetric.** `dataelem.py:689-697`: with a NaN value it calls `math.isnan(other.value)` without checking `other.value` is a scalar (`a == b` → `TypeError: must be real number, not MultiValue`, `b == a` → `False`); the ndarray branch calls `len(other.value)` (`TypeError` for `None`) and the reverse comparison returns an array. Related upstream: the NaN branch was added by PR #2296 (for #2273); commit #2374 made `__eq__` return `bool` but not on these paths.
- **C3 — tuples are documented as accepted but not converted.** `dataelem.py:596-600` detects multi-values with `hasattr(val, "append")`, which tuples lack: `DataElement(0x00280010, "US", (1, 2)).value` warns "A value of type 'tuple' cannot be assigned…" and stores the raw tuple; `write_numbers` then fails with `OSError: required argument is not an integer`. The `add_new` docstrings (`dataset.py:495,527`) say "a list or tuple". Use `isinstance(val, list | tuple | MultiValue)`; add a tuple round-trip test through `dcmwrite`. (`tests/test_dataelem.py:143-145` already builds elements from tuples and passes only by identity.)
- **C5 — group length stored as `private_creator`.** `dataset.py:2829-2839` computes `Tag(group, element >> 8)` for every private element; for the creator element `(0009,0010)` that is `(0009,0000)`, so with a group length present `ds[0x00090010].private_creator` becomes `100` (an `int`) and `.name` degrades to `'Private tag data'`. Only look up a creator for `element >= 0x1000`. Adjacent upstream: #2395/#2396 (stale private block cache) — different symptom, same code area; coordinate.
- **C6 — `keyword`/`is_retired` ignore repeater tags.** `dataelem.py:816-817,830-831` check only `dictionary_has_tag`, whereas `name` (`:790`) also checks `repeater_has_tag`: `DataElement(0x60000010, "US", 1).name` → `'Overlay Rows'` but `.keyword` → `''`. Use the same condition; add a `60xx` test. Related upstream: PR #337 fixed `.name` in 2017.
- **C19 — `repeater_has_tag` is true for private tags.** `datadict.py:480-494` masks only the `xx` nibbles, so `(6001,0010)` matches `60xx0010` although `get_entry` rejects it. Add `not BaseTag(tag).is_private`. (Fix together with C6.)
- **C7 — `DT()` accepts trailing garbage.** `valuerep.py:700` `_regex_dt` has no end anchor and is used with `.match()`: `DT("20200101xyz")` is accepted while `TM("1200xyz")` raises. Use `fullmatch(val.rstrip())`; negative tests.
- **C10 — `PersonName(123)` warns then crashes.** `valuerep.py:1547-1557` assumes `str` after `validate_value` merely warned: `ds.PatientName = 123` → `AttributeError: 'int' object has no attribute 'split'`. Raise a `TypeError` up front (as `UID.__new__` does).
- **C13 — `Tag()` input validation.** `tag.py:127-134` `Tag(1.5)` silently truncates to `(0000,0001)`; `Tag(None)` fails with "'>' not supported between 'NoneType' and 'int'". Add an explicit `isinstance(arg, int)` check with a message mentioning tags; document (or reject) the hex-string forms `"0010"` that the docstring (`:58-64`) omits.
- **C12 — `tag_in_exception` loses the real error.** `tag.py:34-37` `raise type(exc)(msg) from exc` assumes a one-argument constructor; `UnicodeDecodeError` needs five → `TypeError: function takes exactly 5 arguments (1 given)`. It is public but has no callers in `src/`. Use `exc.add_note(msg)` on 3.11+ / fall back to re-raising the original, or deprecate it.
- **C23 — `ISfloat.__str__` ignores `original_string`.** `valuerep.py:1298-1338` (no override; compare `IS.__str__` at `:1410`): `str(IS("1.50", validation_mode=IGNORE))` → `'1.5'` while `original_string` (what gets written) is `'1.50'`. Add the overrides.
- **C24 — `DSfloat` never warns in `WARN` mode.** `valuerep.py:1073-1088` validates only under `RAISE`; `DSdecimal` (`:1210-1229`) warns under `WARN`: an 18-character float is silently accepted and later written as an invalid DS. Mirror the `DSdecimal` structure. Related upstream: #1264/#1334 fixed the `RAISE` case; PRs #2389/#2397 are changing DS formatting — read them first.
- **C25 — per-value padding for multi-valued CS.** `values.py:68-70` `multi_string` strips only the last value: `convert_string(b"A \\B ", True)` → `['A ', 'B']` while `convert_text` and `convert_AE_string` give `['A', 'B']`. PS3.5 §6.2 says trailing spaces are not significant for CS. Strip each item.
- **I4 — `config.debug()` stacks handlers.** `config.py:59-63` appends a `StreamHandler` on *every* call (including `debug(False)`); after `debug(True); debug(True); debug(False)` the logger has three handlers and every message prints three times. Keep a module-level handler reference; remove it on `debug(False)`. Related upstream: PR #901 introduced the behaviour.
- **I2 — `FileSet` validates with `assert`.** `fileset.py:2049` (`assert uid.is_valid` in the `UID` setter → bare `AssertionError()`), `:1417-1421` (`descriptor_file_id`), `:1490-1492` (`FileSet.find`: the comparison itself is the assert). Under `python -O` invalid UIDs are accepted and **`find()` matches every instance that merely has the queried elements**. Replace with `ValueError`/explicit comparisons; test `fs.UID = "bad"`.
- **I10 — `FileSet` leaks its `TemporaryDirectory`.** `fileset.py:995-1002` creates one per instance; only `clear()` (`:1241`) releases it, there is no `close()`/context manager, so GC emits `ResourceWarning: Implicitly cleaning up <TemporaryDirectory …>`. Add `__enter__/__exit__`/`close()` (or `weakref.finalize`) and document it.
- **I8 — `read_partial` on a 2–5 byte forced input.** `filereader.py:872` `unpack("<HH2s", fileobj.read(6))` without checking the read length: `dcmread(BytesIO(b"\x08\x00"), force=True)` → `struct.error`. Treat a short read as EOF/`InvalidDicomError`. Related upstream: PR #2394 handles the 32-bit length read in `data_element_generator`, not this spot; #810/#811 fixed the 0-byte case.
- **I9 — deferred read leaks the re-opened file.** `filereader.py:1172-1190` opens with `fileobj_type(filename, "rb")` and closes only on success; if the file changed, the user gets a raw `struct.error` plus `ResourceWarning: unclosed file`. Use `with`; re-raise as an error that names the deferred element. Related upstream: #1609/#1610 fixed the opposite problem (closing the caller's stream).
- **I11 — `bytes` paths.** `fileutil.py:19` `PathType = str | bytes | os.PathLike` and `path_from_pathlike` (`:426-445`) returns `bytes` for a bytes path, but `dcmread`/`dcmwrite` only test `isinstance(fp, str)` → `TypeError: dcmread: Expected a file path, file-like or readable buffer, but got bytes`. Either drop `bytes` from `PathType` or handle it (this also removes the `# type: ignore[arg-type]` at `:443`).
- **I16 — dead `except EOFError` hiding an `UnboundLocalError`.** `filereader.py:928-940` — `read_dataset` already swallows `EOFError` in `WARN` mode, so the branch is only reachable when it re-raises; if an `EOFError` ever escaped, `dataset` at `:943` would be unbound. Remove the `try`/`except` or initialise `dataset`.
- **I22 — `size_in_bytes("1.5")`.** `misc.py:38` tuple-unpack raises "not enough values to unpack" instead of the intended "Unable to parse length with unit …".
- **I21 — `write_numbers` wraps `struct.error` in `OSError`.** `filewriter.py:385-386`: `ds.add_new(0x00280010, "US", 70000); dcmwrite(...)` → `OSError: 'H' format requires 0 <= number <= 65535`. `ValueError` is the conventional type; discuss in an issue first (changing exception types is a compatibility question).
- **I23 — `raw_element_value_retry` drops the encoding.** `hooks.py:398` calls `convert_value(candidate, raw)` without `encoding`, so a retry into a text VR decodes with the default charset.
- **P9 — negative indices in `iter_array`/`iter_buffer`.** `pixels/decoders/base.py:2218-2220,2350` do not validate like `as_array` (`:1364`): `iter_array(indices=[-3])` silently yields frame 7 of 10; `[-1]` gives a confusing reshape error. Validate per index; test next to `test_negative_index` (`tests/pixels/test_utils.py:1541`).
- **P10 — `_validate_buffer` ignores frame count for 1-bit data.** `pixels/encoders/base.py:519-520` expects `frame_length("pixels") / 8` for a *single* frame; a too-short multi-frame packed buffer passes validation and fails inside the plugin with a `RuntimeError` wrapping an `IndexError`. Multiply by `number_of_frames` (as a lower bound; unpacked 1-byte-per-pixel buffers are also accepted). Related upstream: #2134/#2140 fixed the same mistake on the *decode* side (`DecodeRunner._validate_buffer`); #2380 normalises packed 1-bit OW before encoding.
- **P13 — the endianness fallback never happens.** `pixels/utils.py:1689-1700` (`pixel_dtype`) and the same pattern in `processing.py:189-199,390-400,663-673`: `if hasattr(ds, "file_meta")` wins even when `file_meta` has no *Transfer Syntax UID*, so a dataset with `original_encoding == (False, True)` still raises "Unable to determine the endianness". Factor a `_dataset_endianness(ds)` helper that falls back to `original_encoding` (removes ~40 duplicated lines).
- **P16 — `apply_presentation_lut` on a constant image.** `processing.py:486-489` rescales with `arr /= arr.max() / (nr_entries - 1)` after subtracting the min → division by zero, NaNs cast to 0 (a uniform image of value 100 is treated as 0) with numpy warnings. Guard the zero range; use `float64`.
- **P22 — `NameError` instead of `TypeError` without numpy.** `pixels/encoders/base.py:10-13` swallows the numpy `ImportError` but `:327` still evaluates `isinstance(src, np.ndarray)`; RLE/deflate encoding from bytes is supported without numpy, so this path is reachable. Use the `HAVE_NP` pattern from `decoders/base.py`.
- **P20/P21 — typing.** `decoders/base.py:1252` `Decoder.as_array(**kwargs: DecodeOptions)` (and `common.py:691` `set_options`) means "every keyword value is a `DecodeOptions`", so mypy rejects every real call (`as_array(rows=1)`); siblings use `**kwargs: Any`. The proper `Unpack[DecodeOptions]` needs 3.11 (the `# TODO: Python 3.11` at `common.py:690` is still blocked by `requires-python >= 3.10`). Also 8 unused `# type: ignore` (`processing.py:327,333`, `decoders/pillow.py:48,51,86,95`, `common.py:220`, `utils.py:1631`) — `mypy --warn-unused-ignores src/pydicom/pixels` lists them. **I24** — `filewriter.py:488-490` two ignores in `write_string` vanish if the encoded bytes get their own local name.
- **M6 — download retry never fires with `requests`; no timeouts.** `src/pydicom/data/download.py:101-104` wraps `download_with_progress` in `retry((urllib.error.HTTPError, urllib.error.URLError))`, but when `requests` is importable the function uses `requests.get`, whose exceptions are not subclasses of those → a transient failure is raised after one attempt (the urllib path retries four times). Neither `requests.get` nor `urlretrieve` passes `timeout=`, so a stalled connection hangs forever; `retry` reports with `print()` instead of `warn_and_log`. Add `requests.exceptions.RequestException` (import-guarded), pass timeouts, route messages through the logger; unit-test with `requests.get` monkeypatched. Related upstream: #1333 (connection resets, maintainer suggested retries) and #1340 (added the `requests` path — where the retry tuple stopped matching); #2240 improved the failure message only.
- **M7 — leftovers after PR #2390.** `download.py:266-275`: the recursive re-download passes `data_dir` but **not** `url`, so a caller-supplied mirror is silently replaced by the GitHub URL; the junk file is left on disk after the hash error. Forward `url=url`; unlink before raising.
- **M13 — `pydicom show -q` on RT plans.** `src/pydicom/cli/show.py:81,101` access `fraction_group.ReferencedBeamSequence` and `beam.ControlPointSequence[0]` directly while everything else uses `.get()`; a plan without them (brachy-only fraction groups, anonymised plans) raises `AttributeError`. Use `.get(..., [])`; add a unit test calling `quiet_rtplan` on an in-memory dataset.
- **M14 — the CLI phones home for local files.** `cli/main.py:214-227` calls `get_testdata_file(filename)` for *every* filespec "in case the user forgot the prefix", before checking the local path; a local file whose name matches `urls.json` triggers a download attempt (and an offline warning) to enrich an error message that is never shown. Only look up when `prefix == "pydicom"` or after `FileNotFoundError`, with `download=False`. Related upstream: #1865/#1867 fixed an `IndexError` on the same code path (a local file whose name matched a test-data name) without removing the unconditional lookup.
- **M12 — `util/leanread.dicomfile` on deflated files.** `src/pydicom/util/leanread.py:73-80` keeps parsing compressed bytes after the file meta and yields nonsense tags (`(DDED,6ECF)`, VR `b'\xdc\xd6'`) instead of raising; the module also defines four unused `*VRLittleEndian` constants (`:18-21`), has a leftover `# import pdb;pdb.set_trace()` (`:144`), and no `close()`. Raise `NotImplementedError` for `tsyntax.is_deflated` (or wrap in a zlib decompressor), delete the dead bits, add `close()`. Related upstream: #483/#984 settled that `leanread` is experimental and not for user code — so the maintainers may prefer a loud error over a feature; open PR #2362 handles deflated syntaxes in the main reader only.
- **I3 — `EncapsulatedBuffer.read()` is O(n²) in frames.** `src/pydicom/encaps.py:861-883` recomputes `encapsulated_length`/`lengths`/`offsets` (each a fresh pass over every item) up to three times per loop iteration and scans items linearly from 0: 500 frames → 12 ms/MB, 4,000 frames → 102 ms/MB (a plain `BytesIO` reads in ~0.1 ms). Cache in `__init__` and locate items with `bisect`. Good first performance PR: the fix is local and the before/after numbers are easy to show (there is an `asv` benchmark skeleton in `benchmarks/bench_encaps.py` once M15 is done).
- **T2 — lint `tests/` and `doc/`** (five existing violations; see §3.1).
- **T9 — `[tool.pytest.ini_options]` + coverage config in `pyproject.toml`** (see §3.1).

### 5.3 Intermediate changes (medium)

These need a design decision, touch several functions, or need a careful regression test with crafted data. **Open an issue first** describing the problem with your reproducer, propose the fix, and only then code — this is also how you get maintainer attention before investing days.

- **I1 — odd-length *buffered* OB/OW values corrupt the file.** `src/pydicom/filewriter.py:713-717` takes the length of a buffered (`io.BufferedIOBase`) value from `buffer_remaining(...)` *before* `write_OBvalue`/`write_OWvalue` (`:401-402`, `:420-421`) append the `\x00` pad, so the length field says 3 while 4 bytes are written; every following element is misread (`(0042,0012)` disappears, a bogus `(4200,1200)` appears). Plain `bytes` values are fine. Fix: `value_length += value_length % 2` for buffered values; test: odd-length `BytesIO` value followed by another element, round-tripped. Related upstream: #1511/#1539 fixed this for in-memory bytes; PR #2095 added buffered values without an odd-length case. **This is the most serious defect in the report** (silent data corruption on write).
- **P5 — `iter_array` applies J2K/JLS corrections row-by-row for mixed-precision frames.** `pixels/decoders/base.py:2198-2206,2221-2226` call `_apply_j2k_corrections(arr, runner)` / `_apply_jls_sign_correction(arr, runner)` **without `idx`**; `index=None` means "`arr` holds all frames", but here `arr` is one frame and `runner._frame_meta` already holds one entry per yielded frame, so once two frames differ in `j2k_precision`/`j2k_is_signed`, `zip(arr, signs, bit_shifts)` iterates the *rows* of the frame (row 0 shifted with frame 0's shift, row 1 with frame 1's, the rest untouched). `as_array(index=i)` is right. Fix: pass `idx` at the four call sites; test with a stub plugin yielding frames of different precision (no pylibjpeg needed; `TestApplyJ2KCorrections` in `tests/pixels/test_decoder_base.py:2142` shows the pattern).
- **P6 — `get_j2k_parameters` hangs on `LBox == 0`.** `pixels/utils.py:939-946` walks JP2 boxes with `offset += length` and no guard; ISO 15444-1 allows `LBox == 0` ("to end of file") and `LBox == 1` (8-byte `XLBox` follows), so a malformed or truncated JP2 header makes `pixel_array` **loop forever** instead of failing. Handle both cases and `break` for `length < 8`.
- **P11 — `decompress(ds, use_pdh=True)` is inconsistent.** `pixels/utils.py:646-656,723-730`: the `use_pdh` branch calls `ds.convert_pixel_data(decoding_plugin)`, which decides the backend from `ds._pixel_array_opts["use_pdh"]`, not from the kwarg; unless the caller also ran `pixel_array_options(use_v2_backend=True)`, the bytes come from the `pixels` backend (YBR→RGB converted) while the block that updates *Photometric Interpretation* is skipped — the dataset ends up with RGB bytes labelled `YBR_FULL` and every later `pixel_array` double-converts. The other combination leaves stale options and the next `ds.pixel_array` raises `NotImplementedError`. Make the kwarg and the dataset option agree, reset `_pixel_array`/`_pixel_id`, document `use_pdh`. Deprecated path (gone in 4.0), so keep the change minimal.
- **C11 — `Dataset.copy()` shares the element dictionary.** `dataset.py:641-643` is `copy.copy(self)`, which copies `__dict__` shallowly, so `_dict` is the *same object* in both: `ds_copy.PatientID = "B"` adds to `ds`; `del ds_copy.PatientName` removes from `ds`. There is no `__copy__` and no test. Related upstream: #1146/#1151 implemented `copy()` "like `dict.copy()`" — which would create a new container with shared *elements*; sharing the container was never discussed. Proposal: `__copy__` that does `new._dict = dict(self._dict)` and re-points `_private_blocks`; needs a maintainer decision because `FileDataset.__init__` wrapping a `Dataset` relies on the current behaviour.
- **M1 — `import pydicom` creates `~/.pydicom/data`.** `src/pydicom/data/download.py:76,148,214,310` and `data_manager.py:162` evaluate `get_data_dir()`/`get_config_dir()` as *default argument values*, so `pydicom/__init__.py` → `examples` → `data` → `download` runs `mkdir` at import; `HOME=/dev/null python -c "import pydicom"` raises `NotADirectoryError`, and the cache path is frozen at import. Use `None` defaults resolved inside the functions. Touches five signatures; keep behaviour identical otherwise. Related upstream: #2223/#2224 removed the import-time *download* from `pydicom.examples` but left these import-time defaults.
- **M5 — `codes.CID8134` is unusable.** Two keywords (`ArcuateFasciculus` in FMA and NEU, `SuperiorLongitudinalFasciculus` in NEU and SCT) make `Collection.__getattr__` (`codedict.py:173-178`, "Shouldn't happen, but just in case") raise `RuntimeError: Multiple schemes found…`; because `.concepts`, `__repr__`, `__str__` and `__contains__` iterate every keyword, `repr(codes.CID8134)` and `'ArcuateFasciculus' in codes.CID8134` both raise. Fix in the generator (`util/generate_cids`) or in `__getattr__`; add a test sweeping `dir()` of every CID so regeneration catches it. Related upstream: #1388/#2077 fixed a different ambiguity (one concept in nested CIDs) in the old codedict structure.
- **M9 — `codify -p` loses private sequences.** `util/codify.py:197-201,216` writes a private SQ as `ds.Tag00010001 = tag00010001`; `Dataset.__setattr__` stores that as a plain attribute (with a "Camel case attribute … not a known public element keyword" warning), so the recreated dataset has no private sequence despite `-p`. Emit `ds.add_new((0x0001, 0x0001), 'SQ', …)`; extend `tests/test_cli.py::test_codify_command` to `exec` the output. Related upstream: #2399 (open) adds an `add_new()` fallback in `code_dataelem` for standard elements without a keyword but does not touch `code_sequence`'s `seq_keyword = f"Tag{tag:08x}"`; #1670/#1674 and #1874/#1906 fixed earlier "codify output does not recreate the dataset" bugs — coordinate with #2399's author.
- **M22 — make the suite offline-safe.** Wrap reference-file lookups in a helper that `pytest.skip`s *with a clear reason* when `get_testdata_file` returns `None`; mark the two network tests; make `TestExternalDataSource`'s cache-count assertions relative. Pairs with T13 and D4. **Read PR #1801 first** (closed, unmerged): it proposed a `NO_EXTERNAL_DATA=1` switch that skipped ~350 tests and the maintainers declined — they want the external-data tests to stay *required*. So frame this narrowly: a missing file should produce one informative skip (or a single clear failure naming the file and how to fetch it), never an `AttributeError: 'NoneType'`, and never a blanket opt-out.
- **M24 — `decompress()` materialises every frame before checking the size limit.** `pixels/utils.py:686-694` checks `2**32 - 1` only after `arr.tobytes()` on all frames; compute the expected length from Rows/Columns/SamplesPerPixel/BitsAllocated/NumberOfFrames first. Also fixes the 19-second test.
- **T7 — benchmarks.** Decide the fate of `asv` with the maintainers (delete, or port and add a manual workflow).
- **T4/T3 — one documented developer workflow** (`pip install -e . --group dev` / `uv sync`, `pre-commit run --all-files`, `pytest`, `cd doc && make html`) across `CONTRIBUTING.md`, both tutorials, `doc/README.md` and the Makefile.
- **D21/D22/T14 — make the doc examples testable**: fix `doctest_global_setup`, mark placeholder snippets `+SKIP`, regenerate stale outputs, then add `pytest doc --doctest-glob='*.rst'` to CI so the 104 stale outputs cannot come back.

### 5.4 Advanced / senior-level changes (hard, or need maintainer buy-in)

- **T1 — defuse the ruff time bomb.** Replace `extend-select` with an explicit `select` list (one line), then adopt the ~390 rules ruff 0.16 enables by default *deliberately*: triage by rule (I001 import sorting ×148, B018 useless expressions ×110, SIM201/202 ×62, BLE001 blind except ×26, DTZ001 ×25, …), auto-fix what is safe, add per-rule ignores with a justification for the rest. Coordinate with #2365/#2368. This is a senior task because of the judgement involved and the size of the diff — do it in small, rule-by-rule PRs.
- **T10 follow-up — mypy strictness.** After #2376 lands, enable `warn_unused_ignores`, `enable_error_code = ["ignore-without-code"]`, `warn_unreachable`, and clean up the 49 ignores (several are shown above to be removable).
- **Complexity ratchet.** Set `max-complexity = 20`, `max-branches = 25`, `max-statements = 80`, then split `dcmwrite` (`filewriter.py:1134`, 30/33/94) and `_validate_options` (`pixels/common.py:724`, 23/22) — refactors with zero behaviour change, proven by the existing tests.
- **Adopt `uv`** (`uv.lock`, `uv sync` in docs, `setup-uv` in CI). Needs an RFC-style issue: it changes every contributor's workflow.
- **Docs build on GitHub Actions** with OIDC Pages deploy, replacing the personal deploy key, hard-coded committer identity and per-run TeX install in `build_tools/circle/`.
- **`EncapsulatedBuffer` / `encaps` performance** beyond I3, and the `decompress` pre-check (M24) are the entry to performance work; back every claim with `asv` numbers once T7 is settled (upstream PR #2405 is the model).
- **A `pre-commit` hook for docstring markup** (D29/D30) and a `numpydoc validate` run in CI — small code, but you will be proposing project policy.
- **Help ship 3.1.0 / prepare 4.0**: audit that every `# TODO: Remove in v4.0` API emits a `DeprecationWarning` with a replacement named and a test asserting it; write the 4.0 migration guide. This is exactly the kind of cross-cutting, low-glamour work maintainers value most.

---

## 6. Your path from first PR to senior-level contributor

The stages below map directly onto §5. Each has an exit criterion so you know when to move on.

### Stage 0 — Set up so that CI cannot surprise you (one evening)

```bash
git clone https://github.com/<you>/pydicom && cd pydicom
python -m pip install -U pip            # >= 25.1 for --group
pip install -e . --group dev            # pydicom-data, pyfakefs, pytest, pytest-cov, pre-commit
pip install numpy pillow                # the "basic"/Pillow paths
pre-commit install
python -c "from pydicom.data.data_manager import fetch_data_files; fetch_data_files()"   # populate the cache once
pytest tests -q                         # expect ~3850 passed, ~540 skipped
uvx ruff@0.15.12 check src tests doc && uvx --with types-requests mypy@1.20.2   # the *pinned* tool versions
```

Run the tool versions pinned in `.pre-commit-config.yaml` (via `uvx` or `pre-commit run --all-files`), not whatever `pip install ruff` gives you — the difference is 0 vs 227 errors (§3.1). Read `CONTRIBUTING.md`, `.github/PULL_REQUEST_TEMPLATE.md` and `doc/release_notes/v3.1.0.rst` once.

### Stage 1 — First-time contributor (weeks 1–3): learn the loop

**Do:** one documentation PR from §5.1.1 (e.g. D2–D7, the CONTRIBUTING fixes — you have just lived the problem, so your PR summary writes itself), then one or two one-line code fixes with a test from §5.1.4 (P1, P3, I6, C8 are ideal). Open each as a **draft**, fill in the template, write the summary yourself, and mark it ready only when CI is green.

**Skills you practise:** fork/branch/rebase hygiene; the project's test idioms (`pytest.raises(match=…)`, `pytest.warns`, reference files); writing a regression test that fails before the fix; a release-note line with `:issue:`/`:pr:` roles; reading a review calmly and pushing fix-ups without force-pushing someone else's branch.

**Exit when:** two PRs merged and you can run the full checks locally without reading the docs.

### Stage 2 — Early contributor (months 1–2): own a small behaviour end to end

**Do:** two or three items from §5.2. Suggested order: T12 (xdist — small, universally appreciated, teaches pytest collection), C9 or M2 (`__eq__`/`__hash__` — teaches Python's data-model contracts and release notes for behaviour changes), I4 (`config.debug` — teaches the logging module), then M6 (download retry/timeouts — teaches monkeypatching network code and why the untested code is where the bugs are).

**Skills:** writing the *issue* before the PR (reproducer, expected vs actual, proposed fix); backwards-compatibility thinking (why C15 is a docstring fix and I21 needs a discussion); using `git bisect`/`git log -S` to find when a behaviour appeared (several "related upstream" notes above were found that way); reading PS3.5 to justify a change (C25).

**Exit when:** a maintainer has merged a PR of yours that changed behaviour (not just docs/tests) and you wrote its release note.

### Stage 3 — Intermediate (months 2–4): correctness in the hard parts

**Do:** I1 (buffered odd-length write — a real corruption bug with a clean fix), P5 (per-frame corrections in `iter_array`), P6 (JP2 box grammar), then C11 (`Dataset.copy()` — needs a design proposal). Each of these starts with an issue containing your reproducer and a proposed approach; expect a round of design feedback.

**Skills:** reading the DICOM/JPEG 2000 standards precisely enough to cite sections; crafting binary test fixtures by hand (`DicomBytesIO`, stub plugins) rather than adding large files; measuring performance (I3) and presenting before/after numbers; keeping a PR minimal when you can see ten other things to fix in the same file (file them as issues instead).

**Exit when:** you have fixed a bug that required you to *disagree* with the current code's design in writing and win the argument (or lose it gracefully).

### Stage 4 — Advanced (months 4–8): the project's infrastructure

**Do:** T1 (ruff rule adoption, rule by rule), the mypy strictness follow-up, the complexity ratchet with the `dcmwrite` refactor, the offline-safe/doctest-in-CI work (M22, T14), and the `uv`/docs-CI proposals as RFC issues.

**Skills:** changing shared tooling without breaking anyone (small reversible steps, announce in the issue, provide the one-liner everyone must run); writing an RFC that states the problem, options, costs and a recommendation; pairing with maintainers on things they own (release process, CircleCI secrets).

**Exit when:** a tooling or process change you proposed is the project's default.

### Stage 5 — Senior-level behaviours (ongoing)

Being "senior" here is less about harder bugs and more about *leverage*:

- **Review other people's PRs.** Start with the 40 open ones: reproduce the bug they claim to fix, run their tests, check the release note, point out the missing edge case politely. Reviewing is the fastest way to learn the code base *and* the fastest way to become trusted.
- **Triage issues.** Reproduce new reports, add a minimal example, label difficulty, link duplicates (this project has `Difficulty-*` and `needs contributor` labels that nobody is applying — offer to).
- **Write the design docs nobody wants to write:** the 4.0 migration guide; "how the pixels plugin API works" for codec authors (issue #2370 is asking for exactly a documented plugin listing).
- **Guard the contributor experience** you struggled with in Stage 0: every time the docs/tooling drift, fix it the same week.
- **Measure before and after** for anything called "performance", and **cite the standard** for anything called "conformance".
- **Communicate like a maintainer:** short issues with reproducers, PRs that do one thing, release notes for users not developers, and a clear "I propose X because Y; alternatives considered: Z".

### Working-style notes specific to this project

- Maintainers are few and part-time; a well-formed issue with a reproducer often gets a faster answer than a PR that needs design discussion.
- PRs are squash-merged: keep commit messages useful but do not polish history.
- The AI policy is strict and enforced (`.github/PULL_REQUEST_TEMPLATE.md`): use AI to explore, but reproduce, understand and write up yourself — and say so in the template checklist.
- Prefer existing test files (`pydicom.data`, `examples`) and in-memory `DicomBytesIO` fixtures to adding binaries; large data lives in the separate `pydicom-data` repo.
- Every user-visible change needs a line in `doc/release_notes/v3.1.0.rst` under *Changes* or *Fixes*.
- When in doubt whether something is a bug or a design choice (I21, C15, P7, C11), ask in an issue — the answer is itself a contribution to the record.

---

## 7. Appendix

### 7.1 Commands used for the measurements in this report

```bash
# tooling baselines
uvx ruff@0.15.12 check --config pyproject.toml src tests doc --output-format concise
ruff check --config pyproject.toml src tests doc --statistics            # ruff 0.16.9
for v in 0.15.12 0.16.9; do uvx ruff@$v check --isolated --show-settings src/pydicom/errors.py \
  | sed -n '/linter.rules.enabled/,/linter.rules.should_fix/p' | grep -cE "^\s+[a-z-]+ \([A-Z]+[0-9]+\)"; done   # 59 vs 394
uvx --with types-requests mypy@1.20.2        # clean
mypy                                         # 2.3.1: 13 errors
uvx typos --format brief .

# tests and coverage
python -m pytest tests -q --cov=pydicom --cov-report=term-missing:skip-covered --cov-report=json:coverage.json
python -m pytest tests -n 4                  # fails: "Different tests were collected"
PYTHONHASHSEED=1 python -m pytest tests --collect-only -q > a.txt; PYTHONHASHSEED=2 ... > b.txt; diff a.txt b.txt
python -m pytest doc -q --doctest-glob='*.rst'

# docs markup that will not render
rg -n "^\s*\.\.(version|note|code)" src doc examples
rg -n -E "\([0-9A-Fa-f]{4}, [0-9A-Fa-f]{4}\)" --type rst doc/guides doc/tutorials doc/reference | wc -l   # 104
```

### 7.2 Full per-module coverage (local, no codec plugins), lowest first

```
 18.2   77   pixels/decoders/gdcm.py        (plugin missing)
 22.6  133   pixel_data_handlers/gdcm_handler.py
 25.0   60   pixels/encoders/gdcm.py
 30.9   55   pixels/decoders/pylibjpeg.py
 34.8   46   pixels/encoders/pylibjpeg.py
 41.5  106   pixel_data_handlers/pylibjpeg_handler.py
 50.0   24   pixels/decoders/pyjpegls.py
 52.5   40   pixel_data_handlers/jpeg_ls_handler.py
 60.0   20   pixels/encoders/pyjpegls.py
 61.5   26   data/retry.py                  <- real gap
 65.9   44   util/fixer.py                  <- real gap
 71.7  120   data/download.py               <- real gap
 72.4  214   fileutil.py                    <- real gap (I18)
 88.6   88   util/leanread.py
 88.9   90   pixel_data_handlers/pillow_handler.py
 90.6   53   overlays/numpy_handler.py
 92.1  810   pixels/decoders/base.py
 92.9   28   env_info.py
 93.2   73   pixels/decoders/pillow.py
 93.4  634   pixels/utils.py
 93.8   32   pixel_data_handlers/util.py
 93.9  115   cli/main.py
 95.2  124   jsonrep.py
 96.0  253   charset.py
 96.9  159   config.py
 96.9   64   waveforms/numpy_handler.py
 96.9  392   filereader.py
(all other modules >= 97 %; 38 modules at 100 %)
```

### 7.3 Change log

- **v0.1 (2026-10-27)** — initial review at `b218118`.
