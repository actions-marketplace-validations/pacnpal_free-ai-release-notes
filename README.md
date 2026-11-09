# free-ai-release-notes

A reusable GitHub Actions workflow that writes your release **titles** and **release notes** with AI, for free, using OpenRouter's [free models router](https://openrouter.ai/openrouter/free) (`openrouter/free`).

It reads the commits (and, when allowed, the merged pull requests) between the previous tag and the new one, asks a free model to summarise them into grouped, user-facing Markdown notes and a short title, and then either creates/updates the GitHub release for you or hands the title and notes to the rest of your workflow.

```text
v1.4.0: Offline mode and faster sync

This release adds offline support and makes syncing noticeably faster on large projects.

### ✨ New Features
- Work offline; changes sync automatically when you reconnect (#142)

### 🐛 Bug Fixes
- Fix a crash when opening an empty project (#139)

### ⚡ Performance
- Sync large projects up to 3x faster (#140)

**Full Changelog**: https://github.com/you/app/compare/v1.3.2...v1.4.0
```

## Contents

- [Quick start](#quick-start)
- [Setting up OPENROUTER_API_KEY](#setting-up-openrouter_api_key)
- [Usage examples](#usage-examples)
- [Inputs](#inputs)
- [Outputs](#outputs)
- [Permissions](#permissions)
- [How it works](#how-it-works)
- [Free tier limits and fallback](#free-tier-limits-and-fallback)
- [Privacy](#privacy)
- [Troubleshooting](#troubleshooting)
- [Pinning a version](#pinning-a-version)
- [Developing this repository](#developing-this-repository)

## Quick start

1. [Create an OpenRouter API key](https://openrouter.ai/keys) (free account, no credits needed).
2. In **each repository** that will use the workflow, add it as a secret named `OPENROUTER_API_KEY` (see [below](#setting-up-openrouter_api_key)).
3. Add `.github/workflows/release.yml` to that repository:

   ```yaml
   name: Release

   on:
     push:
       tags:
         - 'v*'

   permissions:
     contents: write

   jobs:
     release:
       uses: pacnpal/free-ai-release-notes/.github/workflows/generate-release-notes.yml@main
       with:
         create-release: true
       secrets:
         OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
   ```

4. Push a tag:

   ```bash
   git tag v1.0.0
   ```

   ```bash
   git push origin v1.0.0
   ```

   The workflow creates the `v1.0.0` release with an AI-generated title and notes. The notes also appear in the job summary.

## Setting up OPENROUTER_API_KEY

The key is **required**, and it must be set in every repository that calls the workflow. The workflow stops at its first step with an error explaining what to do if the key is missing.

**From the GitHub UI:** repository **Settings → Secrets and variables → Actions → New repository secret**, name `OPENROUTER_API_KEY`, value your key.

**With the GitHub CLI** (run inside the repository; it prompts for the value):

```bash
gh secret set OPENROUTER_API_KEY
```

Then pass it to the workflow in one of two ways:

```yaml
    secrets:
      OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
```

or forward all of the caller's secrets:

```yaml
    secrets: inherit
```

> An organization secret works too, as long as it is made available to the repository; it is still read as `secrets.OPENROUTER_API_KEY` in the caller.

## Usage examples

Ready-to-copy caller workflows live in [`examples/`](examples):

| File | When it runs | What it does |
| --- | --- | --- |
| [`release-on-tag.yml`](examples/release-on-tag.yml) | A `v*` tag is pushed | Creates the release with AI title and notes |
| [`release-on-publish.yml`](examples/release-on-publish.yml) | A release is published in the UI | Replaces that release's title and notes in place |
| [`manual-release.yml`](examples/manual-release.yml) | Run by hand with a version | Creates a draft release (and tag) from the chosen branch |
| [`use-outputs.yml`](examples/use-outputs.yml) | A `v*` tag is pushed | Only generates text; your own job builds and publishes |

### Create the release on tag push

See [Quick start](#quick-start).

### Rewrite a release you publish by hand

```yaml
on:
  release:
    types: [published]

permissions:
  contents: write

jobs:
  notes:
    uses: pacnpal/free-ai-release-notes/.github/workflows/generate-release-notes.yml@main
    with:
      create-release: true # an existing release is updated, not duplicated
    secrets:
      OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
```

### Release a tag that doesn't exist yet

Pass `tag` (and optionally `target`); the release, and the tag, are created on that commit.

```yaml
on:
  workflow_dispatch:
    inputs:
      version:
        required: true

permissions:
  contents: write

jobs:
  release:
    uses: pacnpal/free-ai-release-notes/.github/workflows/generate-release-notes.yml@main
    with:
      tag: ${{ inputs.version }}
      target: ${{ github.sha }}
      draft: true
      create-release: true
    secrets:
      OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}
```

### Use the title and notes in your own release step

Leave `create-release` off and read the outputs from a later job:

```yaml
jobs:
  notes:
    uses: pacnpal/free-ai-release-notes/.github/workflows/generate-release-notes.yml@main
    secrets:
      OPENROUTER_API_KEY: ${{ secrets.OPENROUTER_API_KEY }}

  release:
    needs: notes
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v5
      - env:
          GH_TOKEN: ${{ github.token }}
          TAG: ${{ needs.notes.outputs.tag }}
          TITLE: ${{ needs.notes.outputs.title }}
          NOTES: ${{ needs.notes.outputs.notes }}
        run: |
          printf '%s\n' "$NOTES" > notes.md
          gh release create "$TAG" --title "$TITLE" --notes-file notes.md
```

This also works with any release action, e.g. `name: ${{ needs.notes.outputs.title }}` and `body: ${{ needs.notes.outputs.notes }}`.

> Pass the outputs through `env:` as shown rather than writing `${{ needs.notes.outputs.notes }}` directly inside a `run:` script: the notes are model-generated text and should never be pasted into shell code.

### Steer the writing

```yaml
    with:
      instructions: |
        Audience is non-technical end users. Keep it under 10 bullets.
        Call out anything that requires a config change in a "Breaking Changes" section.
      title-template: '{title} ({tag})'
```

### Monorepos and multiple tag series

Use `tag-pattern` so the previous tag is picked from the same series:

```yaml
    with:
      tag-pattern: 'api-v*'
```

or set `previous-tag` explicitly.

## Inputs

All inputs are optional.

| Input | Default | Description |
| --- | --- | --- |
| `tag` | triggering tag | Tag to generate notes for. Defaults to `github.ref_name` when the caller runs on a tag push or release event; required otherwise. |
| `previous-tag` | auto | Tag to compare against. Defaults to the closest tag reachable from the parent of the release commit. Empty on a first release (the whole history is used). |
| `target` | `github.sha` | Commit-ish to release when `tag` doesn't exist yet. Ignored if the tag exists. |
| `tag-pattern` | | Glob for picking the previous tag automatically (`git describe --match`), e.g. `v*`. |
| `model` | `openrouter/free` | OpenRouter model ID. Any model works, including paid ones if your account has credits. |
| `instructions` | | Extra instructions added to the prompt. |
| `title-template` | `{tag}: {title}` | Release title format. `{tag}` is the tag, `{title}` the AI title. When the fallback is used, the title is just the tag. |
| `max-commits` | `200` | Maximum commits sent to the model. |
| `max-input-chars` | `60000` | Maximum characters of git context sent to the model. |
| `temperature` | `0.3` | Sampling temperature. |
| `max-tokens` | `4096` | Maximum tokens in the response. |
| `retries` | `3` | Attempts against OpenRouter before giving up. Each attempt may land on a different free model. |
| `use-github-notes` | `true` | Also send GitHub's own generated notes (merged PRs with authors, new contributors) as context. Needs `contents: write`; skipped silently without it. |
| `append-changelog-link` | `true` | Append a **Full Changelog** compare link. |
| `fail-on-error` | `false` | Fail the job if AI generation fails, instead of using the fallback. |
| `create-release` | `false` | Create the release, or update its title and notes if it already exists. Needs `contents: write`. |
| `draft` | `false` | Create the release as a draft (new releases only). |
| `prerelease` | `false` | Mark the release as a prerelease (new releases only). |
| `runs-on` | `ubuntu-latest` | Runner label. Needs `git`, `python3` and, for `create-release`, `gh`. |
| `api-base-url` | `https://openrouter.ai/api/v1` | OpenRouter API base URL. |

### Secrets

| Secret | Required | Description |
| --- | --- | --- |
| `OPENROUTER_API_KEY` | **yes** | Your OpenRouter API key, stored as a secret in the calling repository. |

## Outputs

| Output | Description |
| --- | --- |
| `title` | Release title (after `title-template`). |
| `notes` | Release notes in Markdown. |
| `tag` | Tag the notes were generated for. |
| `previous-tag` | Tag the changes were compared against (empty on a first release). |
| `model` | The free model OpenRouter actually picked, e.g. `google/gemma-4-31b-it:free`. Empty when the fallback was used. |
| `ai-generated` | `'true'` if the notes came from the model, `'false'` if the fallback was used. |
| `release-url` | URL of the created or updated release (only with `create-release: true`). |

## Permissions

The reusable workflow doesn't declare permissions; it runs with what the **caller** grants its `GITHUB_TOKEN`.

| You want | Caller needs |
| --- | --- |
| Generate text only, from git history | `contents: read` |
| Include merged PRs / contributors as context (`use-github-notes`) | `contents: write` |
| Create or update the release (`create-release`) | `contents: write` |

## How it works

1. **Checks** that `OPENROUTER_API_KEY` is set and stops with instructions if not.
2. **Checks out** the calling repository with full history and tags.
3. **Works out the range**: the release tag (or `target` if the tag isn't created yet) and the previous tag.
4. **Collects context**: commit subjects and bodies (merges excluded, up to `max-commits`), a diffstat of changed files, and GitHub's generated notes when permitted, trimmed to `max-input-chars`.
5. **Calls OpenRouter** (`openrouter/free` by default), which routes the request to a randomly chosen free model that supports the features asked for. The model is asked for JSON with a `title` and Markdown `notes` grouped into Breaking Changes, New Features, Bug Fixes, Performance, Documentation and Maintenance, and told not to invent changes. The answer is parsed defensively (code fences and `<think>` blocks are stripped) and validated.
6. **Retries** on rate limits, server errors and unusable answers with backoff (5s, 10s, 20s, ...). If a model rejects JSON mode, later attempts drop it.
7. **Writes outputs** and a preview to the job summary, and with `create-release: true` creates the release (`gh release create`) or edits the existing one (`gh release edit`).

Model output is never printed to the log or executed: it only goes to step outputs, a Markdown file and the job summary.

## Free tier limits and fallback

`openrouter/free` costs nothing, but free models are rate-limited per account (a per-minute limit and a daily request cap that is higher once you've bought any credits; see [OpenRouter's limits](https://openrouter.ai/docs/api-reference/limits)). One release uses one request, plus one per retry, so normal release cadences stay well inside the limits.

Because the router picks a different free model each time, quality and formatting vary. The workflow handles that by validating the answer and retrying. If every attempt fails:

- with `fail-on-error: false` (default) it **falls back** to GitHub's generated notes, or a plain commit list, with the tag as the title, emits a warning, and sets `ai-generated` to `false`; the release still goes out.
- with `fail-on-error: true` the job fails.

If you want consistent output, set `model` to a specific model, e.g. `google/gemma-4-31b-it:free`.

## Privacy

Commit messages, changed file names and PR titles from the release range are sent to OpenRouter and the free model provider it chooses. Some free providers log prompts or may use them for training, which OpenRouter lets you control in your [privacy settings](https://openrouter.ai/settings/privacy). Don't use this on repositories whose commit history must not leave GitHub.

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| `OPENROUTER_API_KEY is not set` | Add the secret to this repository and pass it under `secrets:` (or `secrets: inherit`). Secrets aren't available to workflows triggered from forks. |
| `OpenRouter rejected OPENROUTER_API_KEY (HTTP 401)` | The key is wrong or revoked; create a new one and update the secret. |
| Warnings with HTTP `404` "No endpoints found matching your data policy" | Your OpenRouter privacy settings exclude every free provider. Allow free endpoints in [privacy settings](https://openrouter.ai/settings/privacy). |
| Warnings with HTTP `429` | Free-tier rate limit. Raise `retries`, or wait; the fallback still publishes the release. |
| `No tag to release` | The caller didn't run on a tag. Pass `tag:`. |
| `Resource not accessible by integration` when creating the release | The caller needs `permissions: contents: write`. |
| Notes compare against the wrong tag | Set `tag-pattern` or `previous-tag`. |
| `ai-generated` is `false` | All attempts failed; the job's warnings say why. |

## Pinning a version

`@main` always runs the latest version of this workflow. To control updates, pin a release tag or, most strictly, a commit SHA:

```yaml
    uses: pacnpal/free-ai-release-notes/.github/workflows/generate-release-notes.yml@v1.0.0
```

```yaml
    uses: pacnpal/free-ai-release-notes/.github/workflows/generate-release-notes.yml@<commit-sha>
```

For private or internal repositories, the repository hosting this workflow must be accessible to the caller (public, or shared under **Settings → Actions → General → Access**).

## Developing this repository

- [`.github/workflows/generate-release-notes.yml`](.github/workflows/generate-release-notes.yml) is the reusable workflow. All logic lives inline, so callers don't need to check out this repository.
- [`.github/workflows/release.yml`](.github/workflows/release.yml) releases this repository with its own workflow on every `v*` tag (needs the `OPENROUTER_API_KEY` secret here too).
- [`.github/workflows/lint.yml`](.github/workflows/lint.yml) runs [actionlint](https://github.com/rhysd/actionlint) over the workflows and examples.

## License

[MIT](LICENSE)
