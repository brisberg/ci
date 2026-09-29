# brisberg/ci

Shared GitHub Actions workflows for `@brisberg` projects.

Two different things live here and they are **not** interchangeable:

- **Callable workflows** in [`.github/workflows/`](.github/workflows) are used *by
  reference* with `jobs.<id>.uses`. One copy exists, it is upgraded once, and every
  caller gets the upgrade.
- **Templates** in [`workflows/`](workflows) and [`experimental/`](experimental)
  predate `jobs.uses` and are meant to be *copied* into a repo. Every copy then rots
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
    uses: brisberg/ci/.github/workflows/twine-pages.yml@v1
```

That is the entire per-game pipeline. No inputs are needed: `spindle.yml` already
names the story and its story format, so nothing game-specific is duplicated into CI.

| Input | Default | Why you would change it |
|---|---|---|
| `tweego-version` | `2.1.1` | Tweego release to build with. Bump here once, for every game. |
| `tweego-sha256` | checksum of 2.1.1 | Must change together with the version — release assets are mutable, so a tag alone is not a pin. |
| `node-version` | `20` | Escape hatch. spindle builds on gulp 4; if a future Node breaks it, one game can pin back without blocking the others. |

Output: `url` — where the game was published.

#### What the caller must have

- **`@brisberg/spindle` >= 0.3.0.** Versions up to 0.2.2 run
  `go run github.com/tmedwards/tweego` rather than executing `tweego`, so they
  ignore the binary this workflow installs. That path is also no longer viable:
  tweego tags `v2.x` while its `go.mod` declares no `/v2` module path, so the Go
  module proxy cannot serve v2.1.1 at all. The prebuilt release archive is the only
  way to pin real tweego, which is why 0.3.0 execs a binary.
- **`permissions` declared in the caller**, exactly as above. A called workflow's
  token can never exceed what the caller granted, so this file cannot grant them to
  itself. Omitting the block fails the deploy, not the build.
- **Pages source set to "GitHub Actions"** in the game repo's settings. On "Deploy
  from a branch" the deploy silently does nothing. Setting it also creates the
  `github-pages` environment that the deploy job references.

## Versioning

Callers pin a tag, never a branch:

```yaml
uses: brisberg/ci/.github/workflows/twine-pages.yml@v1
```

`v1` is a **moving tag**, re-pointed at every backwards-compatible change, so a
tweego or Node bump reaches all callers without editing any of them. That single
upgrade point is the reason this repo exists. Breaking changes get `v2`.

```sh
git tag -f v1 && git push -f origin v1
```

A caller referencing a tag that does not exist fails at workflow *setup* — the run
shows zero jobs and no logs, because nothing was ever scheduled.

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
