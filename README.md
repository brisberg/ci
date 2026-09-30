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
