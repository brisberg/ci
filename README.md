# brisberg/ci

Shared GitHub Actions workflows for `@brisberg` projects.

Two different things live here and they are **not** interchangeable:

- **Callable workflows** in [`.github/workflows/`](.github/workflows) are used _by
  reference_ with `jobs.<id>.uses`. One copy exists, it is upgraded once, and every
  caller gets the upgrade.
- **Templates** in [`workflows/`](workflows) and [`experimental/`](experimental)
  predate `jobs.uses` and are meant to be _copied_ into a repo. Every copy then rots
  independently. Prefer a callable workflow; migrate these as they come up.

## Callable workflows

### `twine-pages.yml` — build a Twine game, deploy it to GitHub Pages

Builds a [`@brisberg/spindle`](https://github.com/brisberg/spindle) project and
publishes the compiled single-file story to the **calling repo's own** Pages site.

```yaml
name: Deploy
on:
  push: { branches: [main] }
  workflow_dispatch:
permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  deploy:
    uses: brisberg/ci/.github/workflows/twine-pages.yml@v2
```

That is the entire per-game pipeline. No inputs are needed: the game's spindle
config names the output file and its `StoryData` passage names the story format, so
nothing game-specific is duplicated into CI.

| Input            | Default           | Why you would change it                                                                                                    |
| ---------------- | ----------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `tweego-version` | `2.1.1`           | Tweego release to build with. Bump here once, for every game.                                                              |
| `tweego-sha256`  | checksum of 2.1.1 | Must change together with the version — release assets are mutable, so a tag alone is not a pin.                           |
| `node-version`   | `22`              | Escape hatch. spindle requires Node >= 22.17; if a future Node breaks a game, it can pin back without blocking the others. |

Output: `url` — where the game was published.

#### What the caller must have

- **`@brisberg/spindle` >= 0.5.1, installed with npm.** The workflow runs
  `npm ci`, then `npm run build -- --out <path>` so that CI, not the game's config,
  decides where the compiled story is written. The repo needs a committed
  `package-lock.json` resolving spindle to 0.5.1 or later (0.5.0 rejects `--out`),
  and a `build` script whose **last** command is `spindle`, since npm appends the
  extra arguments to the end of the script. `-c` is fine. Games still on 0.3/0.4
  (`spindle.yml`, yarn) must stay on `@v1` until they upgrade.
- **Story formats committed to the game repo** in `storyformats/`. spindle 0.5
  no longer bundles any. The formats shipped inside the tweego zip are on hand as
  a fallback, but they change whenever `tweego-version` is bumped, so relying on
  them means a tweego bump can silently change a game's format version.
- **`permissions` declared in the caller**, exactly as above. A called workflow's
  token can never exceed what the caller granted, so this file cannot grant them to
  itself. Omitting the block fails the deploy, not the build.
- **Pages source set to "GitHub Actions"** in the game repo's settings. On "Deploy
  from a branch" the deploy silently does nothing. Setting it also creates the
  `github-pages` environment that the deploy job references.

### `twine-test.yml` — build a Twine game and run its tests

Read-only companion to `twine-pages.yml`. It needs only `contents: read`, so it can
run on every branch. It runs `npm ci`, `npm run build`, then `npm test --if-present`.
If the game depends on `@playwright/test`, Chromium and its system libraries are
installed first, with browsers cached per Playwright version.

Call both workflows from **one** caller workflow, so the deploy waits for the tests:

```yaml
name: CI
on:
  push:
  workflow_dispatch:
jobs:
  test:
    permissions: { contents: read }
    uses: brisberg/ci/.github/workflows/twine-test.yml@v2
  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    permissions: { contents: read, pages: write, id-token: write }
    uses: brisberg/ci/.github/workflows/twine-pages.yml@v2
```

Don't split this into a `test.yml` (every push) and a `deploy.yml` (main). Separate
workflows run in parallel, so `main` would deploy whether or not its tests pass.
Job-level `permissions` keep `pages: write` away from the test job.

| Input            | Default           | Why you would change it                                                                |
| ---------------- | ----------------- | -------------------------------------------------------------------------------------- |
| `tweego-version` | `2.1.1`           | Keep in step with `twine-pages.yml`.                                                   |
| `tweego-sha256`  | checksum of 2.1.1 | Changes together with the version.                                                     |
| `node-version`   | `22`              | Same escape hatch as `twine-pages.yml`.                                                |
| `artifacts-path` | _(empty)_         | Directory to upload when tests fail, e.g. Playwright traces (`test-output/results`). |

#### What the caller must have

- Everything `twine-pages.yml` requires for the build (spindle >= 0.5.1, npm, a
  committed lockfile, `storyformats/`).
- **No placeholder `test` script.** npm's default
  `"echo \"Error: no test specified\" && exit 1"` exists, so `--if-present` runs
  it and it fails. Delete it from games without tests.
- A `test` script that is self-contained: it may rebuild the game, and it must
  find `tweego` on `PATH`, which this workflow provides.

The tweego install step is duplicated from `twine-pages.yml` rather than shared
through a composite action. A reusable workflow can only reference an action at a
pinned ref, not "whatever ref I was called at", so sharing it would couple the two
workflows' releases. Bump both together.

## Versioning

Callers pin a tag, never a branch:

```yaml
uses: brisberg/ci/.github/workflows/twine-pages.yml@v2
```

`v2` is a **moving tag**, re-pointed at every backwards-compatible change, so a
tweego or Node bump reaches all callers without editing any of them. That single
upgrade point is the reason this repo exists. Breaking changes get the next major.

```sh
git tag -f v2 && git push -f origin v2
```

`v1` is frozen: it builds spindle 0.3 (`spindle.yml`, yarn, Node 20). `v2` requires
spindle >= 0.5.1 and npm.

A caller referencing a tag that does not exist fails at workflow _setup_ — the run
shows zero jobs and no logs, because nothing was ever scheduled.

### Tags are repo-wide

A tag points at a commit, and `uses: brisberg/ci/.github/workflows/<name>.yml@v2`
resolves the **whole repo** at that commit. So every callable workflow shares one set
of version tags:

- A workflow that did not change between `v1` and `v2` is byte-identical at both.
  Nothing breaks; the version number just says nothing about that workflow.
- A breaking change to any one workflow forces a new major for all of them.
- Fixing a workflow for old callers means patching a release branch for that major.
  Once `v3` exists, a fix on `main` reaches `v3` only, so `v2` needs cherry-picks and
  a re-pointed `v2` tag.
- Release notes list changes to workflows the reader does not use.

This is fine for a handful of workflows that change together. It degrades as workflows
accumulate and move at different cadences.

### If this repo outgrows a single version line

If the repo gets large, or its workflows become independent of each other, stop
sharing one version line. Two options, in order of effort:

1. **Per-workflow tag prefixes** in the same repo, e.g. `twine-pages/v2` and
   `wiki-publish/v1`, referenced as `…/twine-pages.yml@twine-pages/v2`. Each workflow
   gets its own moving major tag and its own breaking-change history. Do this before
   there are many callers: moving existing callers off bare `v2` means editing every
   one of them, or keeping the old tags alive indefinitely.
2. **Split into separate repos** when workflows are effectively separate products,
   with different audiences, release cadences or owners. This costs more overhead but
   is the only fully clean isolation. It also keeps shared scripts or composite
   actions from coupling every workflow's release to each other.

## Templates (legacy, copy-paste)

Superseded by the callable form above, kept until each is migrated.

- Publish to a GitHub Wiki
  [[doc](workflows/publish-wiki.md) | [workflow](workflows/publish-wiki.yml)] —
  not for `brisberg.dev`, whose CLAUDE.md D9 rejects wiki mirroring.
- Test and Lint with Yarn
  [[doc](workflows/yarn-test-lint.md) | [workflow](workflows/yarn-test-lint.yml)]
- Create Release, with artifacts
  [[doc](workflows/create-release.md) | [workflow](workflows/create-release.yml)]
- [`experimental/`](experimental) — release/upload-asset variants and a yarn
  build-test-lint job set.
