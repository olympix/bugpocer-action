# Olympix BugPocer

## Overview

The Olympix BugPocer action runs Olympix's AI-powered agentic security analysis on Solidity smart
contracts directly inside your pull-request workflow. BugPocer is the agentic counterpart to
Olympix's [Integrated Security](https://github.com/olympix/integrated-security) action: instead
of detector-based static scanning, it dispatches an autonomous auditor that reads the PR diff,
ranks the highest-impact functions, generates and validates proof-of-concept exploits, and posts
its findings back to the PR as review comments on the lines that introduced them.

## Features

- **PR-native UX:** Findings are posted as review comments on the exact lines that introduced
  the vulnerability, with PoC code attached.
- **Slash-command control:** Reviewers can start, cancel, or get help on a scan from the PR
  comment box (`/bugpocer scan`, `/bugpocer cancel`, `/bugpocer help`) — no need to re-trigger
  CI manually.
- **Two trigger modes:** `manual` (recommended — scans only when explicitly requested) or
  `every-commit` (scans every push to a PR).
- **Proof-of-concept generation:** Each finding ships with a Foundry PoC that reproduces the
  exploit so reviewers can verify before merging.

## Getting Started

1. Add a GitHub repository secret named `OLYMPIX_API_TOKEN` containing your Olympix API token.
2. Grant the workflow `pull-requests: write` and `issues: write` permissions so BugPocer can
   post review comments and slash-command replies.
3. Add the `olympix/bugpocer-action` step to your workflow (see examples below).
4. **Manual mode only:** post `/bugpocer scan` as a PR comment, or include `/bugpocer scan` in
   a commit message and push.

## Slash Commands

When `OLYMPIX_GITHUB_TRIGGER_MODE` is set to `manual`, the following commands can be posted as
PR comments — or, for `scan`, included anywhere in a commit message body — to control BugPocer:

| Command | Effect |
|---|---|
| `/bugpocer scan` | Start a BugPocer scan on this PR. One scan may run at a time per PR. |
| `/bugpocer cancel` | Stop the scan currently running on this PR. |
| `/bugpocer help` | Post a help message listing the available commands. |

The parser is case-insensitive on both the prefix and subcommand. Commands inside fenced
` ``` ` code blocks or quoted (`>`) reply lines are ignored, so you can mention them in
documentation without accidentally firing them.

## Usage

### Example 1: Manual mode (recommended)

Runs BugPocer only when a reviewer posts `/bugpocer scan` on a PR comment or pushes a commit
whose message contains `/bugpocer scan`.

```yaml
name: BugPocer Security Analysis (manual)
on:
  issue_comment:
    types: [created]
  pull_request:
    types: [opened, synchronize]

permissions:
  contents: read
  pull-requests: write
  issues: write

jobs:
  bugpocer-analysis:
    if: |
      github.event_name == 'pull_request' ||
      (github.event.issue.pull_request != null &&
       startsWith(github.event.comment.body, '/bugpocer'))
    runs-on: ubuntu-latest
    steps:
      - name: Resolve PR head SHA and number
        id: pr
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          if [[ "${{ github.event_name }}" == "pull_request" ]]; then
            echo "sha=${{ github.event.pull_request.head.sha }}" >> "$GITHUB_OUTPUT"
            echo "number=${{ github.event.pull_request.number }}" >> "$GITHUB_OUTPUT"
          else
            PR_NUMBER=${{ github.event.issue.number }}
            SHA=$(gh api repos/${{ github.repository }}/pulls/$PR_NUMBER --jq .head.sha)
            echo "sha=$SHA" >> "$GITHUB_OUTPUT"
            echo "number=$PR_NUMBER" >> "$GITHUB_OUTPUT"
          fi

      - uses: actions/checkout@v3
        with:
          ref: ${{ steps.pr.outputs.sha }}
          submodules: recursive

      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1

      - name: Run forge install
        run: forge install

      - name: Capture commit message (pull_request only)
        if: github.event_name == 'pull_request'
        run: |
          MSG=$(git log -1 --pretty=%B "${{ steps.pr.outputs.sha }}")
          {
            echo "OLYMPIX_TRIGGER_COMMIT_MESSAGE<<EOF"
            echo "$MSG"
            echo "EOF"
          } >> "$GITHUB_ENV"

      - name: BugPocer Security Analysis
        uses: olympix/bugpocer-action@create_action
        env:
          OLYMPIX_API_TOKEN: ${{ secrets.OLYMPIX_API_TOKEN }}
          OLYMPIX_GITHUB_COMMIT_HEAD_SHA: ${{ steps.pr.outputs.sha }}
          OLYMPIX_GITHUB_PR_MODE: "true"
          OLYMPIX_GITHUB_TRIGGER_MODE: "manual"
          GITHUB_PR_NUMBER: ${{ steps.pr.outputs.number }}
          GITHUB_REPOSITORY_ID: ${{ github.repository_id }}
        with:
          args: -w . -ca
```

### Example 2: Every-commit mode

Runs BugPocer automatically on every commit pushed to a PR, with no slash-command gate.

```yaml
name: BugPocer Security Analysis (every commit)
on:
  pull_request:
    types: [opened, synchronize]

permissions:
  contents: read
  pull-requests: write
  issues: write

jobs:
  bugpocer-analysis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          ref: ${{ github.event.pull_request.head.sha }}
          submodules: recursive

      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1

      - name: Run forge install
        run: forge install

      - name: BugPocer Security Analysis
        uses: olympix/bugpocer-action@create_action
        env:
          OLYMPIX_API_TOKEN: ${{ secrets.OLYMPIX_API_TOKEN }}
          OLYMPIX_GITHUB_COMMIT_HEAD_SHA: ${{ github.event.pull_request.head.sha }}
          OLYMPIX_GITHUB_PR_MODE: "true"
          OLYMPIX_GITHUB_TRIGGER_MODE: "every-commit"
          GITHUB_PR_NUMBER: ${{ github.event.pull_request.number }}
          GITHUB_REPOSITORY_ID: ${{ github.repository_id }}
        with:
          args: -w . -ca
```

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `OLYMPIX_API_TOKEN` | Yes | Your Olympix API token. Store as a repository secret. |
| `OLYMPIX_GITHUB_PR_MODE` | Yes | Set to `"true"` to enable PR mode. |
| `OLYMPIX_GITHUB_TRIGGER_MODE` | Yes | `"manual"` (slash-command gate) or `"every-commit"`. |
| `OLYMPIX_GITHUB_COMMIT_HEAD_SHA` | Yes | The PR head commit SHA the scan should analyze. |
| `GITHUB_PR_NUMBER` | Yes | The PR number being scanned. Resolved differently for `pull_request` vs `issue_comment` events — see Example 1. |
| `GITHUB_REPOSITORY_ID` | Yes | The numeric repository ID (`${{ github.repository_id }}`). |
| `OLYMPIX_TRIGGER_COMMIT_MESSAGE` | Manual-mode `pull_request` events only | Captured from `git log -1 --pretty=%B`. Lets the trigger parser read the commit body for `/bugpocer scan`. |

## Action Inputs

The action accepts a single `args` input that is forwarded to the Olympix CLI. The most common
flags are:

- `-w, --workspace-path` — Project root directory. *Default:* current directory.
- `-ca, --context-aware` — Use the BugPocer agent (required when running in PR mode).

## Workflow Permissions

The job's `permissions` block must include:

- `contents: read` — to check out the repository.
- `pull-requests: write` — to post BugPocer's review comments on the PR diff.
- `issues: write` — to post slash-command replies (`/bugpocer scan` acknowledgments,
  `/bugpocer help` output, cancel confirmations).

## How It Works

BugPocer runs a multi-stage agentic pipeline against the PR's contracts:

1. **Project context build** — extracts invariants, protocols, dependencies, and threat-model
   signals from the codebase. BugPocer may ask you to verify context items and answer security questions.
2. **Code ranking** — scores functions, contracts, and libraries by impact so the highest-risk
   units get analyzed first.
3. **Vulnerability detection** — specialized agents reason about each unit.
4. **PoC generation** — generates Foundry proof-of-concept exploits that reproduce confirmed
   findings.
5. **PR reporting** — posts each finding as a review comment on the diff, with the PoC code and
   remediation guidance attached.

## Documentation

Full documentation is available in the [Olympix User Guide](https://olympix.github.io/Github%20Actions/bugpocer/).

## Support Contact

If you have any questions, feedback, or need help, feel free to contact us at
[contact@olympix.ai](mailto:contact@olympix.ai).
