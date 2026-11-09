# Agent instructions

This repository is a GitHub Action (`action.yml`, logic in `src/generate_notes.py`) that writes release titles and notes with OpenRouter's free models. It is also available as a reusable workflow (`.github/workflows/generate-release-notes.yml`, a thin wrapper that runs the action). See the README for how it works.

## Checks before pushing

- Lint the workflows and examples with [actionlint](https://github.com/rhysd/actionlint), as `.github/workflows/lint.yml` does: `actionlint .github/workflows/*.yml examples/*.yml`.
- `python3 -m py_compile src/generate_notes.py` for syntax; don't commit `__pycache__/`.
- `.github/workflows/test.yml` runs the action and the reusable workflow against this repository on every push and pull request. It needs the `OPENROUTER_API_KEY` secret.

## Releasing

**Release only when the user explicitly asks you to.** Never release on your own initiative: not after merging a pull request, not after fixing a bug, not to "test" the release workflow. A release publishes to everyone using `@v1` and to the GitHub Marketplace.

When asked:

1. **Pick the version.** Use the one the user gave. Otherwise, look at the latest `vX.Y.Z` tag, propose the next [semver](https://semver.org) version (patch for fixes, minor for features, major for breaking changes), and wait for the user to confirm it.
2. **Run the Release workflow on `main`** with that version. Don't push tags yourself; agent sessions often can't, and the workflow creates the tag.
   - GitHub MCP: `actions_run_trigger` with `method: run_workflow`, `owner: pacnpal`, `repo: free-ai-release-notes`, `workflow_id: release.yml`, `ref: main`, `inputs: {"version": "vX.Y.Z"}`.
   - GitHub CLI: `gh workflow run release.yml --ref main -f version=vX.Y.Z`.
3. **Check the run** (Actions > Release, about 2 minutes) and report to the user:
   - the release URL;
   - whether the notes are AI-written: the job log says `Generated notes with <model>`, or warns that fallback notes were used;
   - that `v1` (the major tag) moved to the new release. Prereleases such as `v1.2.0-rc.1` don't move it.
4. **Remind the user about the Marketplace.** Listing a release on the GitHub Marketplace is a checkbox on the release's edit page, **Publish this Action to the GitHub Marketplace**, that only the user can tick. There is no API for it.

To **regenerate the title and notes of an existing release** (for example, after fallback notes were used), run the same workflow with that release's version when the user asks. It keeps the existing tag and rewrites the release in place. Don't delete releases or tags unless the user asks for that specifically.
