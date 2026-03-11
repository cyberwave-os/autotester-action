# Autotester GitHub Action

This GitHub Action runs [Autotester](https://github.com/cyberwaveos/autotester) E2E tests in your CI/CD pipeline.

## Features

- 🤖 Run end-to-end tests using natural language steps
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
        uses: cyberwave-os/autotester-action@v0.1.0
        with:
          action-type: "e2e"
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
```

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
        uses: cyberwave-os/autotester-action@v0.1.0
        with:
          action-type: "e2e"
          config-file: "autotester.yml"
          verbose: "true"
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          STARTING_URL: "https://staging.example.com"
```

## Inputs

| Input         | Description                                          | Required | Default          |
| ------------- | ---------------------------------------------------- | -------- | ---------------- |
| `action-type` | Type of tests to run (`e2e`)                         | No       | `e2e`            |
| `config-file` | Path to the YAML configuration file                  | No       | `autotester.yml` |
| `verbose`     | Enable verbose logging output                        | No       | `false`          |

## Environment Variables

| Variable                     | Description                                         | Required |
| ---------------------------- | --------------------------------------------------- | -------- |
| `OPENAI_API_KEY`             | Your OpenAI API key                                 | Yes      |
| `STARTING_URL`               | Override the base URL for E2E tests                 | No       |
| `AUTOTESTER_AUTH_USERNAME`   | Username for HTTP Basic Auth on protected environments | No    |
| `AUTOTESTER_AUTH_PASSWORD`   | Password for HTTP Basic Auth on protected environments | No    |
| `POSTHOG_PERSONAL_API_KEY`   | Posthog personal API key for session replay links on failed tests | No |

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
    host: "https://us.posthog.com"  # or https://eu.posthog.com, or your self-hosted URL
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
        uses: cyberwave-os/autotester-action@v0.1.0
        with:
          action-type: "e2e"
          verbose: "true"
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          POSTHOG_PERSONAL_API_KEY: ${{ secrets.POSTHOG_PERSONAL_API_KEY }}
```

When a test fails, the output will include a shareable recording link:

```
login-test: Failed!
  Comment: The dashboard did not load after login
  Recording: https://us.posthog.com/shared/abc123token
```

The recording URL is also included in the JSON (`e2e.json`) and XML (`e2e.xml`) report outputs, so you can use it in downstream steps (e.g. posting to Slack or adding a PR comment).

### 5. (Optional) Configure step limits and timeouts

By default, Autotester automatically calculates a sensible agent step limit and wall-clock timeout for each test based on its number of steps. This prevents the browser agent from running indefinitely if it gets stuck in a loop.

The defaults are:

- **`max_steps`**: `number_of_steps * 5` (minimum 20)
- **`timeout`**: `number_of_steps * 60` seconds (minimum 180)

You can override these globally or per test in your `autotester.yml`:

```yaml
e2e:
  max_steps: 40       # global default for all tests
  timeout: 300        # global timeout in seconds
  login-test:
    url: "https://staging.example.com"
    max_steps: 25     # override for this test only
    timeout: 180      # override for this test only
    steps:
      - "Navigate to the login page"
      - "Check that the dashboard is displayed"
```

When a test hits either limit, it is marked as failed with a descriptive message (e.g. `"Test timed out after 300s"`). No changes to your workflow file are needed — just update your `autotester.yml`.

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
