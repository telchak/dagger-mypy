# mypy

A [Dagger](https://dagger.io) module, written in Dang, that type-checks your
Python projects with [mypy](https://mypy-lang.org), using each project's own
mypy configuration, dependencies and mypy version.

## Requirements

Requires Dagger v1.0.0-beta.15 or later.

## Install

```sh
dagger install github.com/telchak/dagger-mypy
```

## Projects

The module type-checks each **mypy project** in your workspace separately. A
project is a directory holding a mypy config, in the order mypy itself reads
them:

- `mypy.ini`
- `.mypy.ini`
- `pyproject.toml` with a `[tool.mypy]` table. A `pyproject.toml` without one
  configures nothing for mypy, so it is no project.
- `setup.cfg` with a `[mypy]` section

A project is keyed by its path from the workspace root (`.` for the root).
List the projects visible from where you stand:

```sh
dagger list mypy-projects -a
```

Discovery scans the whole workspace once, reading only file names and config
text; it never runs mypy or installs anything, so listing stays fast. It never
enters `.git`, `.venv`, `venv`, `node_modules`, `site-packages`, `build`,
`dist`, `.tox`, `.nox`, `.eggs`, `.mypy_cache` or `__pypackages__`.

Each project is checked from its own directory:

- When its config names what to check (`files`, `packages` or `modules`), mypy
  runs with no path and the config decides.
- Otherwise mypy checks the project directory (`mypy .`).

Projects nested inside it are excluded there (`--exclude ^<nested>/`) and
checked on their own, with their own config. A command-line `--exclude`
replaces the config's own `exclude`, so the config's literal patterns are
passed along with it: a single-line value in an INI file, a string or an
array of strings in `pyproject.toml`. A multi-line verbose regex is not read.

## Working directory

Which projects you see depends on where you run `dagger`:

- **At a project root:** that project and the projects below it.
- **Inside a project's subdirectory:** the enclosing project, plus any projects
  below where you stand.
- **Outside any project:** the projects below you.

## Dependencies and the mypy version

mypy needs a project's dependencies to see their types: without them, an
import is an error, or with `ignore_missing_imports` an untyped `Any`, and
checks quietly get weaker. So each project gets a virtual environment, created
with uv from the base image's Python:

1. **Where:** dependencies are installed at the nearest directory at or above
   the project whose `pyproject.toml` has a `[project]` table, or the nearest
   one holding a `requirements.txt`, whichever is deeper.
2. **With what:**
   - a `[project]` table: `uv sync`, which honours `uv.lock`, `[tool.uv]`
     sources and indexes, and installs the default dependency groups (`dev`);
   - a `requirements.txt`: `uv pip install -r requirements.txt`;
   - neither: nothing is installed.
3. **Cached:** for a `[project]` table, the dependencies are installed from
   the `pyproject.toml` files, `uv.lock` and `.python-version` alone, so
   editing source does not re-run that step; the project itself is installed
   after the source is mounted. uv's cache is a volume.
4. **Which mypy:**
   1. the `version` setting, when set;
   2. else the mypy the project's own dependencies install, for instance from
      a `dev` dependency group;
   3. else the latest release.

   The output names the version that ran: `+ mypy . (mypy 2.4.0 ...)`.

The default image is `astral/uv:python3.13-bookworm-slim`, glibc rather than
Alpine, since many packages publish no musl wheel. Set `baseImageAddress` to
an image whose Python your project supports.

Failures name the project and the step:
- `install failed (uv sync --no-install-project, exit 1)` with the end of
  uv's output;
- `mypy failed (exit 1)` with the end of mypy's report;
- an image without uv says so.

## Checks

| Address                    | Runs                            |
| -------------------------- | ------------------------------- |
| `mypy/projects/type-check` | mypy in each project (`@check`) |

```sh
dagger check                                  # every check in the workspace
dagger check --mypy                           # every mypy project
dagger check mypy/projects/type-check         # the same, by address
dagger check --mypy --mypy-project=app        # one project (repeatable)
dagger check -l -a --mypy                     # list one line per project
```

The selected projects are checked in parallel, and every failing project is
reported.

Run the check with `dagger check`, in CI especially: `dagger call` on a check
function does not fail the command when the check fails.

| Flag                    | Selects                                |
| ----------------------- | -------------------------------------- |
| `--mypy`, `--by-mypy`   | checks from this module                |
| `--mypy-project=PATH`   | one project (repeatable)               |
| `--mypy-projects`       | every project                          |
| `--check NAME`          | checks with that name, in every module |

## Using it from another module

```dang
let projects = mypy.projects(ws)
projects.keys                                    # [".", "services/api"]
run(projects.batch.typeCheck(ws))                # check every project
run(projects.get(key: ".").typeCheck(ws))        # one project
```

A check called through a dependency returns a `Check` that has not run yet.
Wrap it to run it and raise its failure:

```dang
let run(check: Check!): Void {
  if (check.pass == false) {
    raise check.error.message ?? "check failed"
  }
  null
}
```

## Settings

Set these in your workspace `dagger.toml`:

```toml
[modules.mypy]
source = "github.com/telchak/dagger-mypy"
settings.baseImageAddress = "astral/uv:python3.11-bookworm-slim"  # default: astral/uv:python3.13-bookworm-slim; any image with uv
settings.version = "2.4.0"                                       # default: "" (the project's own, else latest)
settings.installFlags = ["--all-extras"]                         # default: []; appended to uv sync or uv pip install
settings.environment = ["MYPYPATH=stubs"]                        # default: []; KEY=VALUE for the install and mypy
```

Or from the CLI:

```sh
dagger settings mypy version 2.4.0   # set
dagger settings -u mypy version      # unset, back to the default
dagger settings mypy                 # show every setting
```

## Development

The end-to-end checks live in `.dagger/modules/e2e`, against the fixtures in
`.dagger/modules/e2e/fixtures` and the Python package of
[dagger/pytest](https://github.com/dagger/pytest) as a real-world smoke test:

```sh
dagger check
```

On a Windows host, the Dang SDK's `stale` check reports generated files as out
of date even right after `dagger generate`; the same tree passes it from a
Linux client. The e2e checks are unaffected.
