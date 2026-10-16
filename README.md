Integration testing for the SunPy ecosystem
===========================================

[![Integration matrix](https://github.com/sunpy/sunpy-integration-testing/actions/workflows/integration.yml/badge.svg)](https://github.com/sunpy/sunpy-integration-testing/actions/workflows/integration.yml)

Cross-ecosystem integration tests for sunpy and the packages that depend
on it. Individual packages should still test against dev/pre-release
sunpy in their own CI; the goal here is to catch issues that only appear
when many packages are installed together.

The dashboard is published to
[sunpy.github.io/sunpy-integration-testing](https://sunpy.github.io/sunpy-integration-testing/)
after each scheduled run.

This repository only holds the sunpy-specific configuration. The harness
that installs the packages, runs their tests and renders the dashboard is
[integration-dashboard](https://github.com/OpenAstronomy/integration-dashboard),
and the CI runs through that project's reusable workflow. See its README
for how the `stable`, `pre` and `dev` variants choose package versions,
and for the full config format.

How it works
------------

`.github/workflows/integration.yml` runs on a weekly schedule, on
`workflow_dispatch` and on pull requests. It calls the
integration-dashboard reusable workflow at `@main`, which:

1. reads `columns:` from `packages.yaml` and expands it into a job
   matrix, one job per (Python version, variant) column;
2. in each job, installs the system packages listed in the workflow's
   `libraries` input, builds a shared venv with every listed package
   that can be installed without dragging a shared dependency down for
   the others (the rest are reported as held back), and runs `pytest
   --pyargs <module>` for each one;
3. collects the results and publishes the dashboard to `gh-pages`.

Running locally
---------------

```bash
pip install git+https://github.com/OpenAstronomy/integration-dashboard
# uv is required; see https://docs.astral.sh/uv/
# Some packages also need FFTW and ffmpeg, e.g. on Debian/Ubuntu:
#   sudo apt-get install libfftw3-dev ffmpeg

# Run from the root of this repo so conftest.py and sunpy_pytest.ini
# are picked up.
integration-dashboard run --variant stable --python 3.12

# Or a single package, to iterate faster:
integration-dashboard run --variant stable --python 3.12 --packages ndcube

# Or a tier subset (default: all tiers run):
integration-dashboard run --variant stable --tiers coordinated

# Build the dashboard from whatever results/*.json files exist:
integration-dashboard dashboard

# Preview locally:
python -m http.server -d site 8000
```

Results land in `results/<variant>__<python>.json` and the dashboard in
`site/`. Both directories are gitignored.

Configuration
-------------

Everything is configured in `packages.yaml`:

- `test_deps` lists pytest plugins installed in every venv alongside
  pytest (pytest-remotedata, so `remote_data` tests are skipped).
- `dev_index_urls` lists the nightly wheel indexes used by `dev` columns.
- `columns` lists the (Python version, variant) pairs to test, each one
  a dashboard column. Use uv notation for Python, so `"3.14t"` is the
  free-threaded 3.14 build. The CI matrix is generated from this list.
- `packages` lists the packages to install and test, sunpy included.

To add or disable a package, edit its entry in `packages.yaml`. Each
entry takes:

- `pypi_name` (the package's name on PyPI; also used as the row label)
- `tier` (`coordinated`, `affiliated` or `provisional`; used for grouping
  and the `--tiers` filter. Tiers are shown in the order they first
  appear in the file, so keep each tier's packages together)
- `module` (the top-level Python module name, for `pytest --pyargs`)
- `repo_url` (for the `dev` variant install)
- `install_extras` (list, e.g. `[tests]`)
- `extra_deps` (optional list of extra packages to add to the install)
- `pytest_args` (optional list passed through to pytest; use
  `-k "not foo"` to skip tests)

If a package needs a system library or tool, add it to the
`libraries` input in `.github/workflows/integration.yml`.

Triggering a run from GitHub
----------------------------

1. Actions tab -> `integration-matrix` workflow.
2. "Run workflow" dropdown -> green button.

PR previews
-----------

`integration-matrix` also runs on pull requests, with the same column
matrix as the scheduled run. Instead of publishing to `gh-pages`, it
uploads `site/index.html` as a non-zipped artifact, and the companion
`preview-link` workflow attaches a "View dashboard preview" status
check to the commit that opens the rendered page directly.

To keep PR feedback fast, each package is capped at the first 10
collected tests by the repo-root `conftest.py` (which has to live here
because `pytest --pyargs` collects from site-packages). For the same
reason sunpy's own tests are run with `sunpy_pytest.ini`, since sunpy's
pytest config requires plugins that are not installed here. The preview is a
smoke check of layout, install resolution and the workflow itself, not a
full regression signal. A new push to a PR cancels its in-progress run.

`preview-link.yml` must be on the default branch for its `workflow_run`
trigger to fire, and its `workflows:` entry must match the name of
`integration.yml` (`integration-matrix`).
