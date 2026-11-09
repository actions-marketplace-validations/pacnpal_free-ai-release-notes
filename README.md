# free-ai-release-notes

A reusable GitHub Actions workflow that writes your release **titles** and **release notes** with AI, for free, using OpenRouter's [free models router](https://openrouter.ai/openrouter/free) (`openrouter/free`). Prefer a specific model? [Bring your own](#choosing-a-model), with the free router as a fallback.

It reads the commits (and, when allowed, the merged pull requests) between the previous tag and the new one, asks the model to summarise them into grouped, user-facing Markdown notes and a short title, and then either creates/updates the GitHub release for you or hands the title and notes to the rest of your workflow. [Breaking changes always come first.](#breaking-changes)

```text
v1.4.0: Offline mode and faster sync

### ⚠️ Breaking Changes
- The `sync.legacy` option has been removed; delete it from your config (#141)

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
- [Choosing a model](#choosing-a-model)
- [Breaking changes](#breaking-changes)
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

## Choosing a model

The default, `openrouter/free`, costs nothing: OpenRouter picks one of its free models for each request. To use a model you choose, set `model` to any [OpenRouter model ID](https://openrouter.ai/models):

```yaml
    with:
      model: anthropic/claude-sonnet-4.5
```

Paid models are billed to your OpenRouter account and need credits on it.

**Fallbacks.** List several models, separated by commas or new lines, and OpenRouter tries them in order when one fails (an error, downtime, rate limiting or no credits left). Ending the list with `openrouter/free` keeps releases going for free if your preferred model is unavailable:

```yaml
    with:
      model: anthropic/claude-sonnet-4.5, openrouter/free
```

You're billed for whichever model actually answers; the `model` output says which one it was. To guarantee a release costs nothing, list only `openrouter/free` or models ending in `:free`.

**Per-repository default without editing the workflow.** Read the model from a repository or organization [variable](https://docs.github.com/actions/learn-github-actions/variables) and fall back to the free router when it isn't set:

```yaml
    with:
      model: ${{ vars.OPENROUTER_MODEL || 'openrouter/free' }}
```

```bash
gh variable set OPENROUTER_MODEL --body "anthropic/claude-sonnet-4.5, openrouter/free"
```

An empty `model` also means `openrouter/free`.

## Breaking changes

Breaking changes are always the first thing in the notes, above the summary:

- **Detected from commits.** Commits marked as breaking with [Conventional Commits](https://www.conventionalcommits.org/) (`feat!: ...`, `fix(api)!: ...`, or a `BREAKING CHANGE: ...` footer) are collected across the whole release range, even beyond `max-commits`, and handed to the model as changes it must list under **⚠️ Breaking Changes**, with what users need to do to upgrade.
- **Recognised by the model.** The model is also told to treat removed or renamed APIs, options and commands, changed defaults, and dropped platform support as breaking, even without a marker.
- **Enforced after generation.** Whatever the model writes, any section whose heading announces breaking changes ("⚠️ Breaking Changes", "Breaking:", and so on, but not e.g. "Fixes for breaking builds") is moved to the very top, together with its subsections and labels such as a "Migration" heading or `**Migration:**`. If the model left the section out but commits flagged breaking changes, the workflow adds the section itself from those commits.
- **Nothing flagged is dropped.** The model is asked to end each flagged breaking change with its commit hash. Any flagged commit whose hash or subject doesn't appear in the model's breaking section is added to it. If a model ignores the hash instruction this can repeat an item in different words, which is preferable to a missing upgrade warning.
- **Kept in the fallback.** When AI generation fails, the fallback notes also start with a **⚠️ Breaking Changes** section built from the flagged commits.

Marking breaking commits with `!` or `BREAKING CHANGE:` is the most reliable way to make sure nothing is missed.

## Inputs

All inputs are optional.

| Input | Default | Description |
| --- | --- | --- |
| `tag` | triggering tag | Tag to generate notes for. Defaults to `github.ref_name` when the caller runs on a tag push or release event; required otherwise. |
| `previous-tag` | auto | Tag to compare against. Defaults to the closest tag reachable from the parent of the release commit. Empty on a first release (the whole history is used). |
| `target` | `github.sha` | Commit-ish to release when `tag` doesn't exist yet. Ignored if the tag exists. |
| `tag-pattern` | | Glob for picking the previous tag automatically (`git describe --match`), e.g. `v*`. |
| `model` | `openrouter/free` | OpenRouter model ID, or several separated by commas/new lines to fall back in order. Empty means `openrouter/free`. See [Choosing a model](#choosing-a-model). |
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
| `model` | The model that actually answered, e.g. `google/gemma-4-31b-it:free` when the free router picked it. Empty when the fallback was used. |
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
4. **Collects context**: commits flagged as breaking, commit subjects and bodies (merges excluded, up to `max-commits`), a diffstat of changed files, and GitHub's generated notes when permitted, trimmed to `max-input-chars`.
5. **Calls OpenRouter** with the `model` input (`openrouter/free` by default, which routes the request to a randomly chosen free model that supports the features asked for), plus any fallback models you listed. The model is asked for JSON with a `title` and Markdown `notes`: Breaking Changes first, then a summary, then New Features, Bug Fixes, Performance, Documentation and Maintenance. It is told not to invent changes. The answer is parsed defensively (code fences and `<think>` blocks are stripped) and validated.
6. **Retries** on rate limits, server errors and unusable answers with backoff (5s, 10s, 20s, ...). If a model rejects JSON mode, later attempts drop it.
7. **Puts breaking changes first**: see [Breaking changes](#breaking-changes).
8. **Writes outputs** and a preview to the job summary, and with `create-release: true` creates the release (`gh release create`) or edits the existing one (`gh release edit`).

Model output is never printed to the log or executed: it only goes to step outputs, a Markdown file and the job summary.

## Free tier limits and fallback

`openrouter/free` costs nothing, but free models are rate-limited per account (a per-minute limit and a daily request cap that is higher once you've bought any credits; see [OpenRouter's limits](https://openrouter.ai/docs/api-reference/limits)). One release uses one request, plus one per retry, so normal release cadences stay well inside the limits.

Because the router picks a different free model each time, quality and formatting vary. The workflow handles that by validating the answer and retrying. If every attempt fails:

- with `fail-on-error: false` (default) it **falls back** to GitHub's generated notes, or a plain commit list, with the tag as the title, emits a warning, and sets `ai-generated` to `false`; the release still goes out.
- with `fail-on-error: true` the job fails.

If you want consistent output, set `model` to a specific model, e.g. `google/gemma-4-31b-it:free`, or a paid one with `openrouter/free` as a fallback (see [Choosing a model](#choosing-a-model)).

## Privacy

Commit messages, changed file names and PR titles from the release range are sent to OpenRouter and the model provider that serves the request. Some free providers log prompts or may use them for training, which OpenRouter lets you control in your [privacy settings](https://openrouter.ai/settings/privacy). Don't use this on repositories whose commit history must not leave GitHub.

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| `OPENROUTER_API_KEY is not set` | Add the secret to this repository and pass it under `secrets:` (or `secrets: inherit`). Secrets aren't available to workflows triggered from forks. |
| `OpenRouter rejected OPENROUTER_API_KEY (HTTP 401)` | The key is wrong or revoked; create a new one and update the secret. |
| Warnings with HTTP `404` "No endpoints found matching your data policy" | Your OpenRouter privacy settings exclude every free provider. Allow free endpoints in [privacy settings](https://openrouter.ai/settings/privacy). |
| Warning with HTTP `402` | Insufficient credits; the workflow stops retrying and uses the fallback notes. With a paid model, add credits or add `openrouter/free` as a fallback in `model`. If you're already on free models, your OpenRouter balance is negative, which blocks free models too: add credits. |
| Warnings with HTTP `429` | Free-tier rate limit. Raise `retries`, or wait; the fallback still publishes the release. |
| `No tag to release` | The caller didn't run on a tag. Pass `tag:`. |
| `Resource not accessible by integration` when creating the release | The caller needs `permissions: contents: write`. |
| Notes compare against the wrong tag | Set `tag-pattern` or `previous-tag`. |
| A breaking change isn't listed | Mark the commit with `!` (e.g. `feat!: ...`) or a `BREAKING CHANGE:` footer; flagged commits are always listed. |
| `Release notes exceeded GitHub's 125,000-character limit` | The notes were cut at a line boundary to fit GitHub's release body limit; breaking changes are at the top, so they're kept. Lower `max-commits` or narrow the range with `previous-tag`. |
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
