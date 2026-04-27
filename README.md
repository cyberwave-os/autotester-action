# Autotester GitHub Action

This GitHub Action runs [Autotester](https://github.com/cyberwaveos/autotester) E2E tests in your CI/CD pipeline.

## Features

- 🤖 Run end-to-end tests using natural language steps
- 🎥 Get an MP4 video recording of every test (uploaded as a workflow
  artifact by default) so you can replay exactly what the agent did
- 🐛 Catch regressions early with automated browser flows
- ⚡ Keep E2E checks simple in pull requests and releases

## Usage

### Basic usage

```yaml
name: Run Autotester E2E

on:
  pull_request:
    branches: [main]

jobs:
  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Run Autotester E2E Tests
        uses: cyberwave-os/autotester-action@v0.3.0
        with:
          action-type: "e2e"
          openai-api-key: ${{ secrets.OPENAI_API_KEY }}
```

> **Recordings as artifacts** — by default the action uploads
> `.autotester/` (per-test MP4 videos + JUnit XML + JSON report) as a
> workflow artifact named `autotester-results`. See the
> [Video recordings](#5-video-recordings) section for how to disable this
> or push the videos elsewhere using the action's outputs.

> **Tip — prefer `with:` over `env:`**
>
> Step-level `env:` set on the step that calls a composite action does NOT
> reliably propagate into the composite action's `${{ env }}` context. For
> that reason, from `v0.1.10` onwards we recommend passing secrets and URLs
> as `with:` inputs (`openai-api-key`, `base-url`, `auth-username`,
> `auth-password`, `posthog-api-key`). The older `env:`-based usage still
> works when you set the variables at the job/workflow level or via
> `$GITHUB_ENV`.

### Example: run against staging after deployment

```yaml
name: Autotester E2E

on:
  workflow_dispatch:
  release:
    types: [published]

jobs:
  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Run Autotester E2E Tests
        uses: cyberwave-os/autotester-action@v0.3.0
        with:
          action-type: "e2e"
          config-file: "autotester.yml"
          verbose: "true"
          openai-api-key: ${{ secrets.OPENAI_API_KEY }}
          base-url: "https://staging.example.com"
```

## Inputs

| Input             | Description                                                                                         | Required | Default          |
| ----------------- | --------------------------------------------------------------------------------------------------- | -------- | ---------------- |
| `action-type`     | Type of tests to run (`e2e`)                                                                        | No       | `e2e`            |
| `config-file`     | Path to the YAML configuration file                                                                 | No       | `autotester.yml` |
| `verbose`         | Enable verbose logging output                                                                       | No       | `false`          |
| `openai-api-key`  | OpenAI API key. Alternative to setting `OPENAI_API_KEY` via `env:`.                                 | No       | `""`             |
| `base-url`        | Base URL to combine with relative test URLs. Alternative to `AUTOTESTER_BASE_URL`.                  | No       | `""`             |
| `auth-username`   | HTTP Basic Auth username. Alternative to `AUTOTESTER_AUTH_USERNAME`.                                | No       | `""`             |
| `auth-password`   | HTTP Basic Auth password. Alternative to `AUTOTESTER_AUTH_PASSWORD`.                                | No       | `""`             |
| `posthog-api-key` | Posthog personal API key for session replay links. Alternative to `POSTHOG_PERSONAL_API_KEY`.       | No       | `""`             |
| `model`           | LLM model used to drive the browser agent (e.g. `gpt-5.4-mini`). Alternative to `AUTOTESTER_MODEL`. | No       | `""`             |
| `upload-artifacts`| Upload `.autotester/` (videos + JUnit XML + JSON report) as a workflow artifact. Set to `"false"` to disable and upload the `recordings-path` output yourself. | No | `"true"` |
| `artifact-name`   | Name of the workflow artifact created when `upload-artifacts` is `"true"`.                          | No       | `"autotester-results"` |
| `artifact-retention-days` | Days to retain the workflow artifact. Empty = use repository default.                       | No       | `""`             |

## Outputs

The action exposes the paths it produced as step outputs so you can
post-process them (e.g. upload to S3, attach to a Slack message, comment
on a PR):

| Output            | Description                                                                  |
| ----------------- | ---------------------------------------------------------------------------- |
| `recordings-path` | Directory containing per-test MP4 recordings (one MP4 per test).             |
| `results-path`    | The full `.autotester/` directory (videos + JSON + XML reports).             |
| `results-json`    | Path to `.autotester/e2e.json`.                                              |
| `results-xml`     | Path to `.autotester/e2e.xml` (JUnit format).                                |

## Environment Variables

| Variable                   | Description                                                         | Required |
| -------------------------- | ------------------------------------------------------------------- | -------- |
| `OPENAI_API_KEY`           | Your OpenAI API key                                                 | Yes      |
| `AUTOTESTER_BASE_URL`      | Override the base URL for E2E tests                                 | No       |
| `AUTOTESTER_AUTH_USERNAME` | Username for HTTP Basic Auth on protected environments              | No       |
| `AUTOTESTER_AUTH_PASSWORD` | Password for HTTP Basic Auth on protected environments              | No       |
| `POSTHOG_PERSONAL_API_KEY` | Posthog personal API key for session replay links on failed tests   | No       |
| `AUTOTESTER_MODEL`         | LLM model to use for the browser agent (defaults to `gpt-5.4-mini`) | No       |

## Prerequisites

Autotester uses [browser-use](https://github.com/browser-use/browser-use) for browser automation via Chrome DevTools Protocol. The action handles installing Chrome and browser-use's browser binaries automatically — no extra setup is needed on your part.

If you're running Autotester **locally** (outside of this action), make sure you have Chrome or Chromium installed. You can use the browser-use CLI to install Chromium:

```bash
pip install autotester
browser-use install
```

## Setup

### 1. Create a configuration file

Create an `autotester.yml` file in the root of your repository:

```yaml
e2e:
  # max_steps: 50    # Optional: global agent step limit (default: steps * 5, min 20)
  # timeout: 600     # Optional: global wall-clock timeout in seconds (default: steps * 60, min 180)
  login-test:
    url: "https://your-app-url.com"
    steps:
      - "Navigate to the login page"
      - "Enter 'testuser' in the username field"
      - "Enter 'password123' in the password field"
      - "Click the 'Login' button"
      - "Check that the dashboard page is displayed"
```

### 2. Add your OpenAI API key

Add your OpenAI API key as a secret in your GitHub repository:

1. Go to your repository on GitHub
2. Click "Settings" > "Secrets and variables" > "Actions"
3. Click "New repository secret"
4. Name: `OPENAI_API_KEY`
5. Value: your OpenAI API key
6. Click "Add secret"

### 3. (Optional) Configure HTTP Basic Auth

If your dev/staging environment is protected by HTTP Basic Auth, you can
provide credentials in two ways:

**Option A: Via environment variables (recommended for CI)**

Add `AUTOTESTER_AUTH_USERNAME` and `AUTOTESTER_AUTH_PASSWORD` as repository
secrets, then pass them in your workflow:

```yaml
env:
  OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
  AUTOTESTER_AUTH_USERNAME: ${{ secrets.AUTOTESTER_AUTH_USERNAME }}
  AUTOTESTER_AUTH_PASSWORD: ${{ secrets.AUTOTESTER_AUTH_PASSWORD }}
```

**Option B: Via autotester.yml**

```yaml
e2e:
  auth:
    type: basic
    username: "dev"
    password: "dev123"
  my-test:
    url: "https://staging.example.com"
    steps:
      - "Check the homepage loads"
```

Environment variables always take precedence over YAML values.

### 4. (Optional) Enable Posthog session replay links

If your website uses [Posthog](https://posthog.com) with [session replay](https://posthog.com/docs/session-replay) enabled, Autotester can include a direct link to the recording of each failed test. QA engineers can click the link and watch exactly what the browser did during the failure.

**Step 1:** Add a `posthog` block to your `autotester.yml`:

```yaml
e2e:
  posthog:
    project_id: "12345"
    host: "https://us.posthog.com" # or https://eu.posthog.com, or your self-hosted URL
  login-test:
    url: "https://staging.example.com"
    steps:
      - "Navigate to the login page"
      - "Enter credentials and click Login"
      - "Check that the dashboard is displayed"
```

**Step 2:** Create a [Posthog personal API key](https://us.posthog.com/settings/user-api-keys) with the `session_recording:read` and `sharing_configuration:write` scopes, and add it as a repository secret named `POSTHOG_PERSONAL_API_KEY`.

**Step 3:** Pass the secret in your workflow:

```yaml
name: Autotester E2E with Posthog Replay

on:
  pull_request:
    branches: [main]

jobs:
  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Run Autotester E2E Tests
        uses: cyberwave-os/autotester-action@v0.3.0
        with:
          action-type: "e2e"
          verbose: "true"
          openai-api-key: ${{ secrets.OPENAI_API_KEY }}
          posthog-api-key: ${{ secrets.POSTHOG_PERSONAL_API_KEY }}
```

When a test fails, the output will include a shareable recording link:

```
login-test: Failed!
  Comment: The dashboard did not load after login
  Recording: https://us.posthog.com/shared/abc123token
```

The recording URL is also included in the JSON (`e2e.json`) and XML (`e2e.xml`) report outputs, so you can use it in downstream steps (e.g. posting to Slack or adding a PR comment).

### 5. Video recordings

Every test produces an MP4 video of the browser session, captured locally
via [browser-use](https://github.com/browser-use/browser-use)'s built-in
CDP screencast recorder. No third-party service or signup is required —
the recording happens right inside the GitHub runner.

By default the action uploads the entire `.autotester/` directory
(per-test videos + the JSON and JUnit XML reports) as a workflow artifact
named `autotester-results`. You can download it from the run summary
page in the GitHub UI, or via `gh run download`.

```yaml
- name: Run Autotester E2E Tests
  id: autotester
  uses: cyberwave-os/autotester-action@v0.3.0
  with:
    action-type: "e2e"
    openai-api-key: ${{ secrets.OPENAI_API_KEY }}
```

If you'd rather upload the videos somewhere else (S3, GCS, Slack, a PR
comment, …), turn off the built-in upload and use the action's outputs:

```yaml
- name: Run Autotester E2E Tests
  id: autotester
  uses: cyberwave-os/autotester-action@v0.3.0
  with:
    action-type: "e2e"
    openai-api-key: ${{ secrets.OPENAI_API_KEY }}
    upload-artifacts: "false"

- name: Push videos to S3
  if: always()
  run: |
    aws s3 cp \
      "${{ steps.autotester.outputs.recordings-path }}" \
      "s3://my-bucket/runs/${{ github.run_id }}/" \
      --recursive
```

The recordings directory layout is:

```
.autotester/
├── e2e.json               # full test results, including video_path per test
├── e2e.xml                # JUnit XML
└── videos/
    ├── login-test/
    │   └── login-test.mp4
    └── checkout-test/
        └── checkout-test.mp4
```

> **Note** — Posthog session replay links (the `posthog-api-key` input)
> remain supported and complement the local MP4. PostHog is great for
> watching the user-side experience on a deployed environment; the local
> MP4 covers everything the agent saw in the runner, which is especially
> useful for debugging headless-only failures.

### 6. (Optional) Configure step limits and timeouts

By default, Autotester automatically calculates a sensible agent step limit and wall-clock timeout for each test based on its number of steps. This prevents the browser agent from running indefinitely if it gets stuck in a loop.

The defaults are:

- **`max_steps`**: `number_of_steps * 5` (minimum 20)
- **`timeout`**: `number_of_steps * 60` seconds (minimum 180)

You can override these globally or per test in your `autotester.yml`:

```yaml
e2e:
  max_steps: 40 # global default for all tests
  timeout: 300 # global timeout in seconds
  login-test:
    url: "https://staging.example.com"
    max_steps: 25 # override for this test only
    timeout: 180 # override for this test only
    steps:
      - "Navigate to the login page"
      - "Check that the dashboard is displayed"
```

When a test hits either limit, it is marked as failed with a descriptive message (e.g. `"Test timed out after 300s"`). No changes to your workflow file are needed — just update your `autotester.yml`.

### 7. (Optional) Choose the LLM model

Autotester drives the browser agent with an OpenAI model. By default it uses
`gpt-5.4-mini`. You can override it in three ways (highest precedence first):

1. `AUTOTESTER_MODEL` environment variable
2. `model:` input on this action (forwarded to `AUTOTESTER_MODEL` for you)
3. `model:` key in the `e2e:` section of your `autotester.yml`

```yaml
- name: Run Autotester E2E Tests
  uses: cyberwave-os/autotester-action@v0.3.0
  with:
    action-type: "e2e"
    openai-api-key: ${{ secrets.OPENAI_API_KEY }}
    model: "gpt-5.4-mini"
```

Or set it in `autotester.yml` so the same model is used locally and in CI:

```yaml
e2e:
  model: "gpt-5.4-mini"
  login-test:
    url: "https://staging.example.com"
    steps:
      - "Check the homepage loads"
```

The model name must be one your OpenAI account is entitled to call; Autotester
forwards it verbatim to OpenAI via `browser-use`.

## Supported Languages

Autotester currently supports:

- Python
- TypeScript

## Resources

- [Autotester GitHub Repository](https://github.com/cyberwaveos/autotester)
- [Autotester Documentation](https://docs.autotester.com)
- [Discord Community](https://discord.gg/4QMwWdsMGt)

## License

This project is licensed under the MIT License. See `LICENSE` for details.
