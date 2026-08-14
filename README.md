# `JosiahSiegel/umactually-action`

Run UmActually as a Composite GitHub Action. The action owns Node.js 24 setup, `npm install -g umactually`, first-run secret bootstrap, the live PR review, and verdict output for branch protection.

## Pin to a commit SHA

For supply-chain integrity, pin `uses:` to a full 40-character commit SHA. Floating `@v1` accepts any future tag the action repo publishes; a compromised repo gets to run arbitrary code in your workflow with `pull-requests: write`.

```yaml
- uses: JosiahSiegel/umactually-action@317613abd39061d90f761e965dde1dee8f705e19  # v1
```

Update the SHA in your copy to match the latest tagged release:

```bash
git ls-remote https://github.com/JosiahSiegel/umactually-action.git refs/tags/v1
```

To keep the pin current without hand edits, enable [Dependabot's `github-actions` ecosystem](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuring-dependabot-version-updates#enabling-github-actions-version-updates) on your workflow file. Dependabot opens PRs that bump the SHA whenever a new release ships.

## One-line install

```yaml
name: PR review
on:
  pull_request:
    branches: [main]
    paths:
      - "**.ts"
      - "**.tsx"
      - "**.js"
      - "**.jsx"
      - "**.mjs"
      - "**.cjs"
      - "**.py"
      - "**.go"
      - "**.rs"
      - "**.java"
      - "**.kt"
      - "**.swift"
      - "**.rb"
      - "**.sh"
      - "**.yml"
      - "**.yaml"
      - "**.toml"
      - "**.json"
      - "Dockerfile"
      - "!.github/workflows/pr-review.yml"
concurrency:
  group: umactually-${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true
permissions:
  contents: read
  pull-requests: write
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: JosiahSiegel/umactually-action@317613abd39061d90f761e965dde1dee8f705e19  # v1
        with:
          cli-version: 0.9.3
          provider: openai-compatible
          api-url: ${{ secrets.UMACTUALLY_API_URL }}
          api-key: ${{ secrets.UMACTUALLY_API_KEY }}
```

Secrets are forwarded via the `with:` inputs (`api-url`, `api-key`). Composite Actions cannot access the `secrets.` context directly; the `secrets:` block on `uses:` is a JavaScript/Docker action pattern and is not honored here.

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `cli-version` | no | `__UMACTUALLY_VERSION__` | `umactually` CLI version to install via `npm install -g`. The placeholder `__UMACTUALLY_VERSION__` is substituted by `umactually init` at wizard time; pin a specific tag (e.g. `0.9.3`) for hand-written files. |
| `provider` | no | `openai-compatible` | Provider family: `openai-compatible`, `anthropic`, or `copilot`. |
| `model` | no | `""` | Provider-specific model identifier. Empty = let the provider pick its default. |
| `api-url` | no | `""` | Provider API base URL. Forward your repo secret via `with: api-url: ${{ secrets.UMACTUALLY_API_URL }}`. |
| `api-key` | no | `""` | Provider API key. Forward your repo secret via `with: api-key: ${{ secrets.UMACTUALLY_API_KEY }}`. |

(`config-path`, `skip-draft`, `paths-ignore`, `output-artifact` remain declared as inputs for backward compatibility with the wizard template and pre-v0.9.3 examples, but the action does not forward them to the CLI as of v1.0.1. The CLI auto-discovers `umactually.review.json` from cwd; incremental review is per-PR via GitHub thread queries; the `--files` flag and diff's own ignore list handle path filtering.)

## Outputs

| Output | Description |
| --- | --- |
| `verdict` | `success` / `failure` for required status checks. Branch-protection rules branch on this. |
| `inline-thread-count` | Number of inline review threads posted. Zero is a clean review. |
| `review-id` | Opaque run identifier from the CLI's review artifact. |

## First-run secret bootstrap

If `UMACTUALLY_API_URL` or `UMACTUALLY_API_KEY` is empty on an opening/reopening PR, the action queries existing comments for the `<!-- umactually-bootstrap -->` marker and, if absent, posts an idempotent PR comment explaining the two secrets to configure. The action then exits with the typed error `UMACTUALLY_ERR_SECRET_BOOTSTRAP` (3) so a branch-protection rule surfaces the bootstrap requirement as a required check failure.

## Threat model

- **Supply chain**: SHA-pin `uses:` to a full commit SHA. Floating tags (`@v1`) accept any future tag the action repo publishes. Dependabot auto-updates keep the pin current without hand edits.
- **Fork PRs**: The `branches: [main]` filter restricts the trigger to PRs targeting your default branch. Fork PRs targeting their own branches don't trigger the workflow (so no secret leakage to forks). If you need to review fork PRs, use `pull_request_target` with explicit safety guards — see [GitHub's docs](https://docs.github.com/en/actions/using-jobs/using-conditions-for-jobs) on the tradeoffs.
- **Payload bloat**: The `paths:` trigger filter scopes which file changes run the review. Binary files (`data/`, `.pdf`, etc.) and the workflow file itself are excluded by default. Adjust the globs to your repo's reviewable surfaces.
- **Retries**: The `concurrency:` group with `cancel-in-progress: true` cancels any previous run for the same PR before starting a new one. Prevents thundering reviews.

## License

[MIT](LICENSE).
