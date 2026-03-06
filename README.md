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
        uses: cyberwaveos/autotester-oss-action@v0.1.0
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
        uses: cyberwaveos/autotester-oss-action@v0.1.0
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

| Variable         | Description                          | Required |
| ---------------- | ------------------------------------ | -------- |
| `OPENAI_API_KEY` | Your OpenAI API key                  | Yes      |
| `STARTING_URL`   | Override the base URL for E2E tests  | No       |

## Setup

### 1. Create a configuration file

Create an `autotester.yml` file in the root of your repository:

```yaml
e2e:
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
